---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 29528
source: 综合讨论
publishedAt: 2026-07-18
---

## 为什么需要这道护栏？

当 Agent 被赋予执行 shell 命令、读写文件的工具时，它实际上获得了一张通往整个文件系统的“临时通行证”。无论是 OpenClaw 这类本地智能体框架，还是通过 MCP 接入的外部工具，只要工具声明了 `write_file`、`execute_command` 这样的能力，大模型就有可能在幻觉、恶意提示或错误推理下，删除配置文件、覆盖密钥，甚至修改系统脚本。

在工程实践中，完全不暴露文件系统往往不现实——很多自动化任务天然需要操作临时数据、处理上传的文档或生成报告。权衡之下，**为文件访问加上白名单护栏**是成本最低、最务实的防线之一。它不会试图猜测 Agent 想做什么，而只是简单地规定：你只能动这几个目录。

## 问题定义

以 OpenClaw 生态为例，一个典型的 MCP 文件读写工具可能长这样：

```python
@mcp.tool()
def read_file(path: str) -> str:
    with open(path, "r") as f:
        return f.read()
```

问题显而易见：`path` 接受任意字符串，`../.ssh/id_rsa` 或绝对路径 `/etc/passwd` 可以直接穿透当前工作目录。即便 Agent 当前以低权限用户运行，一旦有可读写的敏感目录挂载，后果依然严重。

目标十分明确：在所有文件工具入口处增加一层路径校验，确保操作始终落在预先指定的白名单目录内。如果检查不通过，工具直接返回明确错误，而不是交给操作系统去赌权限。

## 做法：从白名单定义到工具封装

### 1. 声明允许的目录

工程上通常会定义 2–3 个目录作为白名单，例如项目数据目录、临时工作区和一个只读的资源目录。为了消除歧义，所有路径都在声明时立即进行规范化：

```python
import os

ALLOWED_DIRS = [
    os.path.realpath("/app/agent_workspace"),
    os.path.realpath("/tmp/agent_sandbox"),
    os.path.realpath("/data/readonly_resources"),
]
```

`os.path.realpath` 会解析所有符号链接、消除 `..` 并将路径转为绝对路径，这是后续校验的基础保障。

### 2. 实现通用校验函数

核心逻辑：对于输入的任意路径，先解析出真实的绝对路径，然后判断它是否位于任一白名单目录之下。

```python
def safe_path(raw_path: str) -> str:
    target = os.path.realpath(raw_path)
    for allowed in ALLOWED_DIRS:
        # commonpath 在两者完全无关时会抛出 ValueError，需捕获
        try:
            if os.path.commonpath([target, allowed]) == allowed:
                return target
        except ValueError:
            continue
    raise PermissionError(
        f"Path '{raw_path}' resolves to '{target}', which is outside allowed dirs."
    )
```

关键点：
- 使用 `os.path.realpath` 而非 `abspath`，因为 `abspath` 并不解析符号链接，攻击者可以通过链接跳转到白名单之外。
- `commonpath` 比较的是两者共同前缀，必须等于白名单目录本身（而不是某个同名前缀）。例如白名单为 `/app/data`，路径 `/app/data_extra/file` 的共同前缀是 `/app`，不等于 `/app/data`，会被正确拒绝。  
- 同时支持文件和目录操作：commonpath 对 `/app/data/file.txt` 和 `/app/data` 返回 `/app/data`，符合预期。

### 3. 在 MCP 工具中挂载护栏

有了 `safe_path` 之后，每一个文件操作工具只需要在一开始调用它，后续使用返回的绝对路径即可。

```python
@mcp.tool()
def read_file(path: str) -> str:
    allowed = safe_path(path)
    with open(allowed, "r") as f:
        return f.read()

@mcp.tool()
def write_file(path: str, content: str) -> str:
    allowed = safe_path(path)
    # 可选：进一步限定只能写入已有文件或只能在特定子目录创建
    with open(allowed, "w") as f:
        f.write(content)
    return f"written to {allowed}"
```

如果想做得更细，还可以针对读/写分别设定白名单，或对写入操作增加文件后缀检查。但不建议在校验层加入过于复杂的业务规则，否则容易在维护时引入疏漏。

### 4. 集成与测试

在 OpenClaw 的 `config.yaml` 或 MCP server 启动脚本中，将上述工具注册进去，并**立即加入路径穿越的回归测试**：

- 测试 `../../etc/passwd`
- 测试符号链接：先在白名单目录内创建一个指向 `/etc` 的链接，再尝试通过该链接读写
- 测试绝对路径直接指向白名单外的路径
- 测试白名单目录自身的直接操作（如读写 `/app/agent_workspace` 是否会被误杀）

这些用例应该在 CI 中自动化，因为后续任何对白名单目录的修改都可能引入绕过。

## 踩坑经验

1. **符号链接是最大的盲区**  
   如果仅使用 `os.path.abspath` 而不使用 `realpath`，攻击者哪怕只在白名单目录内放一个指向 `/etc` 的快捷方式，就能轻松绕过。`realpath` 是底线。

2. **`commonpath` 的 ValueError**  
   在 Windows 上，不同盘符的路径（如 `C:\` 和 `D:\`）调用 `commonpath` 会抛出 ValueError。必须用 try/except 包裹，否则工具会直接 crash。Linux 下虽然不常见，但挂载点变化也可能触发。工程上统一捕获比较稳妥。

3. **白名单目录本身可能不存在**  
   如果使用容器化或临时文件系统，白名单目录可能尚未创建。此时 `realpath` 会返回不存在的路径，后续 `open` 会得到 FileNotFoundError，误伤正常操作。建议在 Agent 启动时预创建白名单目录，或在校验后、实际操作前创建。

4. **相对路径与工作目录**  
   `safe_path` 内部使用 `realpath` 时，相对路径是基于进程的当前工作目录解析的。如果 Agent 运行过程中改变了工作目录（`os.chdir`），同一个相对路径可能解析到不同位置。解决方案：要么禁止 Agent 变更工作目录，要么在每次调用时传入基准目录并显式拼接。

5. **不要忘记命令执行工具**  
   文件读写只是入口之一。如果 Agent 还能调用 `execute_command("cat /etc/shadow")`，白名单形同虚设。对于 shell 工具，建议进一步结合容器隔离或 AppArmor/SELinux 策略，而非只靠路径检查。

## 可复用建议

- **将校验封装为装饰器或工具中间件**：如果项目内有多个文件工具（read/write/copy/move），重复写 `safe_path` 容易遗漏。可以直接写一个 `@require_whitelist` 装饰器，统一在被装饰函数的第一个路径参数上施加校验。
- **配置化白名单**：不要把白名单写死在代码里，通过环境变量或配置文件注入。例如 `AGENT_ALLOWED_DIRS=/app/data,/tmp/sandbox`，启动时解析并做 `realpath`。
- **记录并监控越权尝试**：当工具抛出 `PermissionError` 时，务必记录原始请求路径和解析结果，这些日志是发现模型行为异常、提示词注入的第一手信号。
- **配合最小权限原则**：即使有了路径白名单，也建议 Agent 进程以专用系统用户运行，文件系统权限设置为只读/只写特定目录，实现纵深防御。
- **额外考虑写入安全**：如果必须允许写入，可叠加文件大小限制、扩展名白名单、不覆盖已有文件等策略，防止 Agent 通过写垃圾数据炸盘。

## 总结

给自动化脚本加本地目录白名单，本质上是**把 Agent 的“能力范围”显式地画在文件系统里**。它不是银弹——真正的安全还需要容器、权限、审计等多层防护，但这一层护栏实施成本极低，却能阻止大量因幻觉、错误推理或提示注入导致的越权访问。在工程化落地的 Agent 系统中，它应该是每一个文件工具出厂前的默认配件。

> 一句话经验：先 `realpath`，再 `commonpath`，不到白名单不入场。

---

