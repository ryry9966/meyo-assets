---
title: 给 Agent 脚本加本地目录白名单：文件访问护栏实操
feedId: 29155
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景：当 Agent 拿到文件系统的钥匙

无论是基于 MCP 的本地工具服务、自动化脚本插件，还是直接运行在设备上的 Agent 进程，一旦具备文件读写能力，就等于拿到了一把高权限钥匙。一个没有护栏的 Agent 可能会在 prompt 偏差、工具链调用链错误或上游幻觉的情况下，意外删除配置文件、覆盖日志，甚至遍历到家目录下的密钥文件。

传统的解决办法是“信任调用方”，但在 Agent 链路里，调用方本身可能由模型生成，参数不可靠。更工程化的做法是：**在文件访问的第一道关口，用目录白名单做强制校验，任何越界操作直接拒绝。**

## 问题定义：不只是“检查一下路径”

如果仅仅判断请求路径是否以白名单目录开头，攻击面会非常大：

- **符号链接绕过**：`/allowed/symlink_to_etc` -> `/etc`
- **路径规范绕过**：`/allowed/../etc/passwd`
- **相对路径绕过**：Agent 切换工作目录后传入 `../../secret`
- **大小写/分隔符差异**：Windows 盘符、 macOS 的 `/private` 映射等
- **TOCTOU 竞态**：检查通过后、操作前，路径被替换为 symlink

护栏要解决的，是在“用户意图送达文件系统”之前，把真实的目标路径限制在可信目录范围内。实现上不是一行 `startswith`，而是一个带有路径解析序的检查函数。

## 做法与步骤

### 1. 定义白名单与统一入口

白名单用绝对路径列表，典型配置如下（可环境变量注入）：

```python
import os
ALLOWED_ROOTS = [
    os.path.realpath("/data/agent_workspace"),
    os.path.realpath("/var/tmp/agent_sandbox"),
]
```

所有 Agent 触发的文件操作（读、写、删除、rename 等）必须通过一个 `safe_path()` 检查函数获得合法路径，否则拒绝。

### 2. 将用户路径转为安全绝对路径

核心思路：先把传入路径变成不含 `..` 和符号链接的 **真实绝对路径**，再判断是否在任一白名单前缀下。

```python
import os, pathlib

def safe_path(user_path: str, base_dir: str | None = None) -> pathlib.Path:
    # 若传入相对路径，基于指定的 base_dir（或固定工作目录）解析
    if not os.path.isabs(user_path):
        if base_dir is None:
            raise ValueError("相对路径必须提供 base_dir")
        user_path = os.path.join(base_dir, user_path)

    # 解析所有符号链接，规范化绝对路径
    real = pathlib.Path(os.path.realpath(user_path))

    # 检查是否在任一白名单根目录下
    for root in ALLOWED_ROOTS:
        try:
            real.relative_to(root)
            return real
        except ValueError:
            continue
    raise PermissionError(f"路径越界: {user_path}")
```

文件操作时，只使用返回的 `safe_path` 对象，不再信任原始输入。

### 3. 集成到 Agent 工具调用

如果是 MCP 服务，在资源端点或工具函数入口处统一调用 `safe_path`：

```python
@mcp.tool()
def read_file(filepath: str):
    path = safe_path(filepath, base_dir="/current/workspace")
    return path.read_text()
```

对于 Python 的 `os`、`shutil`、`open` 等调用，可以封装一个轻量 `SandboxFS` 上下文，或通过全局函数替换，避免漏网之鱼。

### 4. 处理符号链接与挂载点

`os.path.realpath` 会跟随符号链接并解析为最终目标，这一步能防御大部分 symlink 绕过。但对于仍可能存在的边界问题（如 `/proc/self/fd` 等特殊文件系统），可以在白名单检查前附加 `is_symlink` 预警，或者在白名单目录本身使用 `openat2` + `RESOLVE_NO_SYMLINKS` 系统调用（Linux 5.6+）实现原子化路径解析。

## 踩坑记录

- **Windows 盘符与长路径**：`C:\Users` 与 `C:\users` 在 Win32 下不敏感，`os.path.realpath` 对大小写不强制一致，需要额外做 `Path.resolve()` 或 `os.path.normcase` 比较。
- **macOS `/tmp` 变体**：`/tmp` 是 `/private/tmp` 的符号链接，如果白名单写 `"/tmp"`，realpath 会变成 `"/private/tmp"`，此时白名单若未覆盖`/private/tmp` 会导致误判。解决方式：白名单也用 `os.path.realpath` 初始化。
- **相对路径的 base_dir 混乱**：若 Agent 内部会 `os.chdir()`，`safe_path("file.txt")` 的 base 将被更改。应当始终显式传递 `base_dir`，或禁止 Agent 进程内随意切换工作目录。
- **TOCTOU 仍可能被利用**：检查通过到实际 `open` 期间，路径可能被外部进程修改。对于高安全要求场景，应尽量用 `open` + `O_NOFOLLOW` 等 flags，或采用 Linux 的 `openat2`，但会增加可移植复杂度。绝大多数自动化场景下，`realpath` 检查已足够阻断脚本级滥用。
- **白名单目录自身不可写？** 若 Agent 被允许在 `/data/agent_workspace` 内创建 symlink 指向外部目录，然后访问该 symlink，`realpath` 会解析到外部并被拒绝——这正是护栏的效果。但要注意，护栏不能阻止 Agent 在工作目录内部覆盖自己的脚本，需要额外版本控制或只读挂载限制。

## 可复用建议

1. **做成通用中间件**：将 `safe_path` 封装成一个 `PathGuard` 类，挂载在 `open`、`os.remove`、`shutil.move` 等调用前。可以用 `unittest.mock.patch` 或手动注入到 Agent 的全局命名空间，强制所有文件操作走检查。
2. **配置化白名单**：通过环境变量 `AGENT_ALLOWED_ROOTS=/data/workspace:/tmp/sandbox` 注入，启动时解析。对于多租户场景，用进程级隔离避免交叉访问。
3. **日志与告警**：每次越界拒绝都记录请求路径、调用栈和 Agent 任务 ID，便于发现 prompt 注入或 tool 误用。
4. **最小权限原则**：即使有了护栏，仍建议 Agent 以专用系统用户运行，配合文件系统 ACL 双重兜底。
5. **测试用例化**：将边界案例（相对路径、`..`、symlink、换行符空格路径）固化为单元测试，每次修改代码后跑一次验证。

## 总结

给自动化脚本加目录白名单，不只是“加个 if”，而是一个以路径解析为核心的工程习惯。`os.path.realpath` 是成本最低的起步手段，但在多平台和严苛安全要求下需要搭配符号链接预警、固定 base_dir 和可能的原子化系统调用。护栏的目标不是绝对安全，而是让随意读写变成需要刻意绕过才能成功，将意外风险降到可接受范围。

对于 OpenClaw 社区的实践者，如果你的 Agent 已经接入本地文件工具，强烈建议在下一次迭代中加入这一层检查。它会省去很多深夜排查问题时的冷汗。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/25cc1e882c1f35ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/f491b5c35eb579d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/c5d44e93ebabb731.png)

