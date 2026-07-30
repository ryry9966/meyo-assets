---
title: 别让 Agent 拿到你的整个硬盘：给文件操作工具加目录白名单
feedId: 31022
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：Agent 的「全能错觉」

在 OpenClaw、MCP server 或者手搓的 ReAct agent 里，我们习惯于把“读写文件”作为一种基础能力赋予 Agent。比如让它在工作目录下创建配置、读日志、整理输出，甚至操作 Git 仓库。开发阶段环境可控，往往直接给一个 `file_write` 或 `exec_command` 工具，然后就不再细想权限边界。

但当你把同样的 Agent 配置搬到长期运行的自动化任务中，或者允许它处理用户上传的文件名、外部传入的参数时，**没有受控的文件访问立刻变成一个高危弱点**。Agent 不会主动作恶，但它可能被误导删除数据、覆盖敏感配置、或者在无意间把 `.env` / `.ssh` / 密钥文件读出发送到外部。

这不是理论威胁。一个允许写入任意路径的工具，配合一段看似无害的 prompt，足以让一个本地 agent 变得非常危险。

## 问题：默认打开的文件权限有多宽

典型的 agent 工具定义长这样（以 Python 函数工具为例）：

```python
def write_file(path: str, content: str):
    with open(path, "w") as f:
        f.write(content)
```

没有任何过滤。Agent 拿到这个工具后，`../../../`、绝对路径、符号链接、Windows 盘符全都能穿过。即使你企图用 prompt 约束“只能操作当前项目目录”，对大模型来说这只是一句建议，没有强制力。

在 MCP 生态里情况类似：filesystem 相关的 MCP server 往往暴露 `list_directory`、`read_file`、`write_file` 等能力，如果没有参数校验和路径限制，效果等同于 `sudo rm -rf` 包装成了自然语言接口。

需要一个工程化的、能落地的护栏。

## 做法：实现带目录白名单的文件工具

核心思路：**在调用任何文件系统操作前，将用户传入的路径规范化，然后检查它是否落在预先定义的白名单目录内。**如果不在，直接拒绝，返回错误而不是执行。

### 1. 定义白名单目录列表

```python
import os
from pathlib import Path

ALLOWED_ROOTS = [
    Path("/app/workspace"),
    Path("/app/outputs"),
]
```

白名单写死在代码或配置中，不允许通过 prompt 传入，否则又绕开了。

### 2. 路径安全检查函数

```python
def safe_resolve_path(user_input: str) -> Path:
    # 如果传入的是相对路径，相对于当前工作目录解析
    given = Path(user_input)
    if not given.is_absolute():
        given = Path.cwd() / given

    # 关键：resolve() 会消除所有 .. 和符号链接
    resolved = given.resolve()

    # 检查是否落在白名单内
    for root in ALLOWED_ROOTS:
        try:
            resolved.relative_to(root)
            return resolved
        except ValueError:
            continue

    raise PermissionError(f"Access denied: {user_input} resolves to {resolved}")
```

要点：
- 必须用 `resolve()` 消除 `../` 陷阱。
- 不用 `os.path.realpath()` 也可以，`pathlib.resolve()` 效果一致。
- 白名单用 `relative_to()` 检查包含关系，不会误判。

### 3. 封装安全的文件工具

```python
def safe_write_file(path: str, content: str):
    allowed_path = safe_resolve_path(path)
    allowed_path.write_text(content, encoding="utf-8")

def safe_read_file(path: str) -> str:
    allowed_path = safe_resolve_path(path)
    return allowed_path.read_text(encoding="utf-8")
```

现在即使 agent 输入 `../../.env`，`resolve()` 会算出真实路径，一旦跳出 `/app/workspace` 就直接抛出 `PermissionError`，工具调用失败，agent 会收到拒绝的错误信息。

### 4. 集成到 OpenClaw / MCP / 自定义 agent

- **OpenClaw 自定义工具**：直接把 `safe_write_file` 等注册为工具函数，不需要任何底层改动。
- **MCP server**：在 tool handler 里调用同一个 `safe_resolve_path` 做校验，失败返回 error message。
- **Shell 命令权限**：如果你的 agent 允许执行 shell 命令，白名单机制也需要考虑。但建议完全避免直接 shell，改用参数化子进程并同样限定工作目录。

## 踩坑点

1. **符号链接穿越**
   `resolve()` 会跟随符号链接，这点很好。但如果你的白名单目录本身是符号链接，需要提前统一处理。建议白名单路径也用 `Path(...).resolve()` 初始化。

2. **竞态条件（TOCTOU）**
   校验和实际文件操作之间存在时间窗口。在本地 agent 场景一般不是主要威胁，但如果要极致安全，可以在打开文件后再次检查文件描述符的真实路径（Linux 上通过 `/proc/self/fd`）。大多数工程场景靠 resolve 足够。

3. **Windows 盘符与长短路径**
   在 Windows 下要注意盘符大小写和 `~` 短路径问题，`pathlib` 的 `resolve()` 能处理大部分情况，但建议实测一下跨盘符写入。

4. **temp 目录混淆**
   如果允许访问 `/tmp`，注意不同用户、Docker 容器内的共享问题。最好把临时文件限制在某个专用子目录，也纳入白名单。

5. **错误信息不要泄露路径细节**
   生产环境中，`PermissionError` 返回给 agent 的信息不要包含完整路径，避免目录结构枚举。可以简单返回 `"Access to this path is not allowed."`。

## 可复用建议

- **尽早做校验**：所有文件读写、删除、移动、复制都过同一个 gate 函数。
- **白名单最小化**：只给 Agent 任务实际需要访问的目录，不要图方便加根目录。
- **与工作目录隔离**：把 agent 的 CWD 设为白名单内的某个子目录，并用 `Path.cwd() / user_input` 处理相对输入，减少绝对路径的暴露。
- **审计日志**：对拒绝访问的记录加日志，便于排查是 prompt 问题还是异常行为。
- **容器化兜底**：即便有白名单，也可把 agent 运行在容器里并 mount 只读或限定卷，形成纵深防御。

## 总结

给 agent 文件操作加目录白名单，本质上是用**代码强制替换信任**。它不需要复杂的权限系统，几十行逻辑就能把不可控的路径遍历变为可枚举的安全范围。对于任何会运行在本地、拥有读写能力并且接收外部输入的自动化 agent，这是一个投入极小、回报极高的工程习惯。

下次你再给 agent 添加文件工具时，先问一句：如果它试图访问 `~/.ssh/id_rsa`，我的代码会拦住吗？

---

