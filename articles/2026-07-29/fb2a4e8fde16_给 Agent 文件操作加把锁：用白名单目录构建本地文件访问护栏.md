---
title: 给 Agent 文件操作加把锁：用白名单目录构建本地文件访问护栏
feedId: 30917
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 OpenClaw 的自动化实践里，我们经常让 Agent 通过 MCP 文件服务或 Shell 工具去读写本地文件——帮我们整理文档、清洗日志、操作本地仓库。但是，一旦把 `cp`、`cat` 或 `write` 这样的权限直接交给 Agent，隐患就随之而来：模型可能产生意外指令，或者被恶意提示注入，导致它去触碰 `~/.ssh`、环境变量文件，甚至执行破坏性删除。

大多数 MCP 文件服务器（如官方的 `@modelcontextprotocol/server-filesystem`）本身支持通过 `allowedDirectories` 做粗粒度的访问控制，但如果我们在自定义 Skill、插件或自研工具链里做了二次封装，或者需要更细粒度的动态白名单，就得自己实现一道“文件访问护栏”。

## 问题定位

不加限制的文件操作至少带来两类风险：

1. **信息泄露** — Agent 读取 `/etc/passwd`、`~/.aws/credentials` 等敏感文件并返回给模型，后续可能通过会话历史或日志落地。
2. **文件损坏** — Agent 误写或删除关键配置、项目源码，甚至 `rm -rf`。

目标很明确：**让 Agent 只能在我们预先圈定的目录内进行文件读写，任何试图跳出白名单的操作都应被阻断并留下记录。**

## 做法：构建一个带白名单校验的 MCP 文件工具

下面以 Python 实现一个最小可行的受限文件工具为例，展示如何在工具层叠上一层白名单护栏。这个工具可以直接作为 OpenClaw 的 Custom MCP Server 使用。

### 1. 确定白名单目录

以环境变量 `ALLOWED_DIRS` 传入，逗号分隔多个绝对路径，要求目录必须真实存在：

```python
import os
ALLOWED_DIRS = [
    d.strip() for d in os.getenv("ALLOWED_DIRS", "/home/user/agent-sandbox").split(",")
    if os.path.isdir(d.strip())
]
```

### 2. 实现路径检验函数

核心逻辑是：将用户传入的路径解析为**真实的绝对路径**，再判断是否以任一白名单目录开头。为了防止符号链接和 `..` 绕过，必须使用 `os.path.realpath`（或 Python 3.6+ 的 `Path.resolve()`）。

```python
from pathlib import Path

def is_path_allowed(user_path: Path) -> bool:
    # 若传入相对路径，先与默认白名单目录拼接
    if not user_path.is_absolute():
        user_path = Path(ALLOWED_DIRS[0]) / user_path

    # 解析到真实路径：检测符号链接、去除 .. 等
    try:
        real_path = user_path.resolve(strict=False)
    except RuntimeError:
        return False

    # 对于尚不存在的文件/目录，检查其父目录的真实路径
    if not real_path.exists():
        real_parent = real_path.parent.resolve(strict=True)
        return any(
            str(real_parent).startswith(allowed) for allowed in ALLOWED_DIRS
        )

    return any(
        str(real_path).startswith(allowed) for allowed in ALLOWED_DIRS
    )
```

### 3. 封装文件操作并在每步调用前检查

我们实现几个安全的文件原语：`read_file`、`write_file`、`list_directory`。每个方法都先调用 `is_path_allowed`，不放行就直接抛异常或返回错误。

```python
def safe_read_file(filename: str) -> str:
    path = Path(filename)
    if not is_path_allowed(path):
        raise PermissionError(f"Access denied: {filename}")
    # 只读文本文件，避免二进制误操作
    return path.read_text(encoding="utf-8", errors="ignore")

def safe_write_file(filename: str, content: str) -> None:
    path = Path(filename)
    if not is_path_allowed(path):
        raise PermissionError(f"Access denied: {filename}")
    # 确保父目录存在
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding="utf-8")
```

然后将这些函数注册为 MCP 工具。如果你使用 FastMCP，只需加上装饰器即可。

### 4. 接入 OpenClaw

在 OpenClaw 的 `mcp_servers` 配置中指向这个自建 MCP 服务，Agent 就可以调用 `read_file`、`write_file` 等工具，并且所有请求都会被白名单逻辑过滤。

## 踩坑实录

实际部署时，几个坑需要提前填好：

- **`resolve()` 的坑**  
  `Path.resolve(strict=False)` 遇到文件不存在时也能返回“假设的真实路径”，这很有用。但如果路径中包含中间不存在的符号链接，可能仍无法得到正确结果。**建议对不存在的文件，确保父目录真实可解析**，如示例代码所示。对于新建文件，一定先检查父目录的真实性。

- **路径遍历的变种**  
  `../../` 很容易被解析掉，但要注意 `.. ;` 或其他操作系统特有的短名称（Windows 的 `~1` 之类）。统一用 `realpath` 能解决大部分问题，但 Windows 盘符大小写要额外处理：可用 `os.path.normcase(real_path)` 配合白名单的 `normcase` 比较。

- **白名单目录内部的破坏**  
  即便 Agent 只在白名单目录内活动，也仍可能执行 `shutil.rmtree` 批量删除。需要对操作类型做限制，比如禁止递归删除，或要求高风险操作需二次确认（在 Agent 侧由用户确认）。可以通过在 MCP 工具定义中增加一个 `allow_dangerous_ops` 开关，默认关闭。

- **日志与审计**  
  所有被 `is_path_allowed` 拒绝的访问都必须记录，包含时间戳、原始请求路径、解析后的真实路径。这能帮助发现和复盘异常行为。

## 可复用建议

- **将检查逻辑抽象为装饰器或中间件**  
  如果有多组文件工具，用装饰器 `@require_allowed_path` 可以避免代码重复。装饰器内部处理路径解析，并将拒绝行为统一为结构化错误响应。

- **配合容器化与只读挂载**  
  白名单目录内，再通过容器文件系统的权限控制加强隔离。例如：将 Agent 工作目录以只读方式挂载，仅对白名单目录开放写权限。这为护栏提供了第二层防御。

- **动态白名单**  
  对于需要临时授权某个一次性任务访问外部目录的场景，不要直接扩大全局白名单。可以在工具中提供一个 `allow_temp_access(path, ttl)` 机制，同样经过解析和记录，超时后自动吊销。

- **针对 MCP 服务本身的防护**  
  如果你的 MCP 服务进程以较高权限运行，文件护栏就更为关键。在 Docker 中运行该服务，用非 root 用户并限制 `capabilities`，可降低整体风险。

## 总结

给 Agent 的文件操作加上本地目录白名单，投入很小（几十行代码），但对防止“越狱”式文件访问非常有效。核心思路就是**解析真实路径 → 匹配白名单前缀 → 拒绝未知访问**，再配合日志与容器限制。这道护栏并不是银弹，既不能解决 Agent 在白名单内干了蠢事（比如覆盖了白名单里的关键文件），也不能防御内核级提权，但在工程实践中已经足以拦住绝大多数由于模型幻觉或提示注入导致的无意文件泄露。

把文件访问牢牢锁在划定的沙箱里，是 Agent 在生产环境中安全落地的基础一步。

---
*附：如果你正在使用官方 filesystem MCP server，可以直接利用其 `allowedDirectories` 选项实现类似效果，无需自行编码；本文方案适用于需要更灵活控制或二次开发的场景。*

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/14146d20d9546250.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/8b7b0fbe2a96b922.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/59b7a95003dc6752.png)

