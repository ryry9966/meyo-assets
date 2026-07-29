---
title: 为 Agent 自动化脚本添加文件访问护栏：目录白名单实战
feedId: 30912
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 OpenClaw 这类支持工具调用 (Tool Calling) / MCP 的智能体平台里，让 Agent 执行本地脚本或读写文件是常见的自动化需求。想象一个场景：你让 Agent 帮你整理某个下载目录，它会自动遍历文件、重命名、分类移动。脚本逻辑本身没问题，但如果 Agent 错误地将 `rm -rf $HOME/some_important_folder` 当作清理命令，或者因为 prompt 注入被诱导去读取 ~/.ssh/私钥，后果会很严重。

问题在于，**Agent 运行时的文件访问通常继承了进程用户的全部权限**，没有任何边界。一条看似无害的指令可能触及系统关键区域。单纯依赖 prompt 约束不够——工程实践需要更硬的护栏：**目录白名单**。

## 问题拆解

我们要实现的效果是：Agent 调用的任何文件操作工具（读、写、列表、删除）都只能作用在预先声明的目录内，即便传入恶意路径也会被拦截。技术上需要解决这几个点：

1. 路径规范化和解析——防止 `../`、符号链接等绕过。
2. 白名单匹配——精确判断真实路径是否在白名单下。
3. 与 Agent 工具无缝集成——最好是工具内部静默校验，调用方无感知。

以 OpenClaw 生态为例，我们可以自制一个 MCP 服务器，暴露受控的文件操作工具给 Agent。下面给出一个基于 Python 的极简实现思路，可以轻易复现到任何支持自定义工具的平台上。

## 实现步骤

### 1. 设计白名单集合

从安全角度，推荐只开放必要目录，例如：

```python
ALLOWED_ROOTS = {"/data/agent_workspace", "/tmp/agent_sandbox"}
```

可以使用环境变量注入，避免硬编码。

### 2. 编写安全校验函数

核心是路径解析与白名单检查。使用 `os.path.realpath()` 可以解析所有软链接、相对路径、`..` 等，得到“最终真实路径”。然后判断该路径是否以白名单中的某个目录作为前缀。注意，要防止 `realpath` 返回符号链接到白名单外部的情况，所以必须在解析后检查。

```python
import os

def is_path_allowed(path: str) -> bool:
    # 如果路径本身就不存在，realpath 会失败，可 fallback 到 abspath + 递归去掉 .. 等
    try:
        real_path = os.path.realpath(path)
    except Exception:
        # 若目标是创建新文件，realpath 可能报错，可先用 os.path.abspath 并规范化
        real_path = os.path.abspath(os.path.normpath(path))
    # 确保最终路径在任何一个白名单根目录下
    for root in ALLOWED_ROOTS:
        if real_path.startswith(root.rstrip("/") + "/") or real_path == root:
            return True
    return False
```

如果担心时间窗口竞争（符号链接在检查后又被替换），可考虑在操作前再次检查，或者使用 `O_NOFOLLOW` 等系统级标志，但通常工程上先保证解析逻辑正确就够了。

### 3. 封装受控工具

比如我们要提供受控的“读取文件”工具：

```python
def safe_read_file(filepath: str) -> str:
    if not is_path_allowed(filepath):
        raise PermissionError(f"Access denied: {filepath}")
    with open(filepath, "r") as f:
        return f.read()
```

类似地可以为写入、列表、删除等操作加上相同的检查。你可以在 MCP 服务器中暴露这些函数作为工具：

```python
# mcp 服务器伪代码
mcp_server.tool("read_file")(safe_read_file)
mcp_server.tool("write_file")(safe_write_file)
```

### 4. 集成到 OpenClaw Agent

在 OpenClaw 中配置 MCP 客户端，指向你启动的本地 MCP 服务器（通过 stdio 或 SSE）。Agent 工具列表中便会出现 `read_file`、`write_file` 等受控操作。之后无论 Agent 从何处获取路径（用户输入、任务规划、被注入的指令），最终都会经过白名单拦截。

## 踩坑记录

- **路径存在性影响解析**  
  `os.path.realpath` 要求路径真实存在；如果脚本试图创建一个现在还不存在的文件，`realpath` 会抛出异常。需要区分场景：对“写入/创建”操作，可以先对父目录做白名单检查，再允许在父目录下创建新文件。这样就不会绕过。

- **符号链接带来隐患**  
  即使通过了白名单检查，攻击者也可能将白名单内的普通文件链接到外部敏感文件（如 `/etc/passwd`），然后 Agent 读这个文件就会泄露信息。可以在检查阶段同时要求路径所在目录和文件本身都不为符号链接，或者要求在解析后再次确认最终目标在白名单内（已经做），但是若 Agent 有写入权限，可能先创建恶意符号链接再让 Agent 读。更深度的保护可以启动时用 `chroot` 或容器隔绝。

- **Windows 下的路径大小写**  
  如果用 Windows，需要统一到小写或使用 `os.path.normcase`，否则白名单匹配会失效。建议全部使用小写进行比较。

- **`..` 和尾随斜杠**  
  如果白名单是 `/tmp/agent`，恶意路径 `/tmp/agent/..` 可能导致前缀匹配误判。好在 `realpath` 会消除 `..`，但若回退到 `abspath`+`normpath`，必须保证这种组合也被规范化成 `/tmp` 而被拒绝。测试路径：`/tmp/agent/..` → 规范化后应为 `/tmp`，不在白名单，从而拦截。

## 可复用建议

1. **白名单配置化**：通过环境变量或配置文件注入目录列表，避免代码修改。
2. **日志与告警**：记录所有被拒绝的访问请求，便于发现潜在异常或配置错误。
3. **容器化隔离**：在最外层使用 Docker/Bubblewrap 将整个 Agent 运行时目录挂载为只读或受限，实现纵深防御。
4. **定期审计**：检查白名单目录是否被意外扩大，或是否有脚本试图越界。
5. **限制操作类型**：根据需求，只暴露必要操作（如只读文件、只允许追加写入等），减少攻击面。

## 总结

给 Agent 脚本加本地目录白名单，本质上是将“原则上的安全”落实为可执行的代码约束。通过路径解析、白名单前缀检查、同时防范符号链接和竞争条件，可以切实降低 Agent 误操作或恶意注入带来的破坏。这套模式不限于 OpenClaw，任何允许运行自定义自动化脚本的智能体平台都能迁移使用。Agent 安全的起点，就是把每一次文件访问都当成不可信的外部输入来对待。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/74fb54decbd65fa7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/84d54939325988e7.png)

