---
title: 给 Agent 的文件操作加把锁：本地目录白名单护栏实践
feedId: 29479
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景：Agent 正在触碰你的文件系统

随着 Agent 框架（OpenClaw、MCP 宿主等）让 LLM 能直接调用本地工具，文件读写已经从“人来操作”变成“模型可触发”。常见场景：Agent 根据上下文生成配置文件、整理日志、批量重命名、执行代码解释器输出的脚本。安全边界一旦模糊，模型在一个遭注入的对话里就可能无意间覆盖 `/etc/hosts`、读取 `~/.ssh/id_rsa`，甚至通过符号链接跳转到敏感路径。

简单说：你给了 Agent 一把仓库钥匙，但仓库里还放着保险柜——我们需要它只碰指定的货架。

## 问题拆解：给定一套文件系统工具，如何收敛可访问范围？

假设 Agent 运行时能够调用一个 `write_file(path, content)` 或 `execute_command("cat ...")` 接口。我们无法穷举所有危险路径，更无法在 prompt 里靠“道德约束”让模型自觉。工程上要的是**硬护栏**：无论 prompt 被怎么诱导，工具底层拒绝超出白名单的路径。

需求细化：
- 允许访问指定目录树，比如 `/home/user/project/data/` 及其子目录。
- 白名单可能有多个目录，需要支持多条目。
- Actor 在调用工具时，路径参数可能来自模型生成，可能包含 `..`、符号链接、冗余分隔符等绕过手段。
- 拒绝时返回清晰错误，方便 Agent 自纠正或终止任务。

## 做法：内置一个轻量路径校验器

以下是基于 Python 的典型实现思路，可直接嵌入 OpenClaw 的自定义工具或 MCP 服务器的资源端点。

### 1. 定义白名单配置
以列表形式放到环境变量或 Agent 运行配置中，方便运维修改：
```python
import os

ALLOWED_ROOTS = os.getenv("FS_WHITELIST_ROOTS", "/var/agent/sandbox,/tmp/agent").split(",")
```
为保证安全，约定所有路径都用绝对路径，杜绝“相对路径依赖当前工作目录”引发的问题。

### 2. 路径规范化与校验函数
核心逻辑：对传入的用户路径做解析，解决 `..`、符号链接，然后判断规范后的真实绝对路径是否以任意一个白名单根目录开头。

```python
import os
from pathlib import Path

def is_allowed_path(user_path: str, extra_roots: list[str] | None = None) -> tuple[bool, str]:
    """校验路径是否在白名单内，返回 (是否允许, 解析后绝对路径)"""
    roots = ALLOWED_ROOTS + (extra_roots or [])
    # 1. 粗粒度过早预防：禁止空或 null 字节
    if not user_path or "\x00" in user_path:
        return False, ""
    # 2. 绝对化并解析符号链接
    given = Path(user_path).expanduser()
    try:
        # resolve 会跟随符号链接并返回绝对路径
        real = given.resolve(strict=False)
    except (OSError, RuntimeError):
        return False, ""
    # 3. 二次保障：逐级检查父目录是否可信任? 可选
    # 对于一些通过 mount 绕过的场景需要进一步处理，这里先保留基本判断
    # 4. 白名单匹配
    for root in roots:
        root_path = Path(root).expanduser().resolve(strict=False)
        try:
            real.relative_to(root_path)
            return True, str(real)
        except ValueError:
            continue
    return False, str(real)
```

### 3. 在工具函数内集成
在 `write_file`、`read_file` 这类暴露给 Agent 的工具中，第一步就校验：

```python
def agent_write_file(path: str, content: str) -> dict:
    allowed, real_path = is_allowed_path(path)
    if not allowed:
        return {"error": f"Access denied: {path} is outside allowed directories."}
    # 后续操作全部基于 real_path
    Path(real_path).parent.mkdir(parents=True, exist_ok=True)
    Path(real_path).write_text(content)
    return {"status": "ok", "written_to": real_path}
```

对于 `execute_command` 类工具，由于命令可以是 `cp /etc/passwd /allowed_dir/` 等多种形式，仅靠路径白名单无法覆盖。建议把这部分额外封装，除非 Agent 确实需要 shell 能力，否则用原子化文件工具取代命令行操作。

## 踩坑点记录

1. **符号链接绕过**：用户创建一个指向 `/etc` 的符号链接放在白名单目录内，然后通过它访问敏感文件。`Path.resolve()` 会跟随 symlink，所以真实路径就暴露在白名单之外，校验失败。这恰恰是我们希望的——默认拒绝。但要小心，某些实现先访问再检查会导致 TOCTOU 竞态。解法：永远先校验再操作，且使用 `resolve()` 拿到真实路径。

2. **已有的文件描述符传递**：如果 Agent 运行环境里还有别的进程共享文件描述符，攻击面会扩大。本文限定在 Agent 自身的工具调用入口处拦截，不解决进程间传递问题。推荐搭配容器/沙箱使用。

3. **相对路径依赖**：如果忘记在函数内部 `expanduser()` 和转为绝对路径，模型传一个 `../../` 就能跳出去。强制在一开始就做 `resolve()` 是防呆设计。

4. **路径分隔符差异**：Windows 和 Linux 分隔符不同，但 `pathlib` 已能自动处理。要小心的是配置里的白名单，如果用系统环境变量保存，比如 `C:\agent\sandbox` 在 Linux 上解析会异常。统一用 `Path` 处理可缓解。

5. **动态添加白名单**：有些场景要求 Agent 在执行过程中临时申请目录（比如用户说“把文件存到 ~/Downloads”）。该需求需要额外的审批流，单纯静态白名单不够。可按需实现一次性的“用户确认授权”，但不要做成模型可控的配置项，否则护栏形同虚设。

## 可复用的工程建议

- **配置分层**：将白名单根目录通过配置文件注入，在 Agent 启动时固化，运行期不可变。避免运行时通过工具修改白名单，除非有独立的 authz 机制。
- **审计日志**：每次工具调用，无论成功失败，都记录 `user_path`、`resolved_path`、调用时间、是否允许。审计是最后一道防线。
- **与容器能力组合**：将 Agent 进程本身放在只读根文件系统的容器里，再把白名单目录以 `tmpfs` 或 bind mount 方式挂入。这样即使护栏代码有缺陷，攻击者也无法突破容器限制。简单说：护栏 + 沙箱双保险。
- **测试用例**：建议写一批边界测试覆盖：深层 `..`、`/./` 冗余、空字符串、`~`、符号链接环、超长路径等。我踩过的坑里，一半是路径规范化没处理好的 corner case。
- **MCP 场景**：如果是用 MCP 暴露文件系统资源，建议将白名单校验放在 server 端 tool handler 入口，而不是依赖客户端传入的枚举列表。保证暴露出去的能力本身就是受约束的。

## 总结

给 Agent 加文件访问护栏，本质上是在模型与操作系统之间引入一个不可绕过的最小权限层。这层代码可能短短几十行，但它补上了“LLM 生成任意路径”这一最大的安全敞口。实现上注意路径解析的完备性，结合容器隔离，能把普通的自动化脚本风险降到可接受范围。不要因为“只是个内部工具”就跳过这一步——工程习惯往往是在这里分化的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/283478ec5cec3782.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/6916484b3066c1aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/857429372c1aad09.png)

