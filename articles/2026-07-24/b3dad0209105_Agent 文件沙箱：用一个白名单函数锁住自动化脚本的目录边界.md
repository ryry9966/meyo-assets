---
title: Agent 文件沙箱：用一个白名单函数锁住自动化脚本的目录边界
feedId: 30300
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 MCP 工具、Agent 插件或自动化脚本里，我们经常需要让大模型读写本地文件。典型场景包括：总结某个项目目录下的代码、批量处理 Markdown 文档、读取配置文件后执行下一步操作。一旦给了文件读写能力，Agent 就有了“触角”，但也就引入了风险。如果没有明确约束，一个误生成的路径或恶意注入的指令就可能删掉 `~/.ssh`，或者读取 `/etc/passwd`。

于是，与其事后补救，不如在文件访问层加一道护栏——**目录白名单**。思路很简单：只允许 Agent 在预先声明的几个目录及其子目录下操作，其他路径一律拒绝。相比黑名单和动态权限提示，白名单对自动化流程最友好，接口清晰，实现代价小。

## 问题拆解

我们想要的效果是：给定一个即将读/写的目标路径，判断它是否落在白名单目录内。理想状态下，这个判断要做到：

- 防止路径穿越 (`../../etc/passwd`)
- 处理符号链接 (symlink 跳出允许目录)
- 兼顾相对路径与绝对路径
- 在不同操作系统上行为一致

需要注意，**单纯做字符串前缀匹配是不安全的**。`/home/user/allowed/../secret/file` 用前缀匹配可能被误判为在 `/home/user/allowed` 下，实际上它指向了 `/home/user/secret/file`。因此需要先对路径做规范化，再去比较。

## 做法：一个可复用的路径检查函数

下面是一个基于 Python `pathlib` 的实现，可以直接嵌入 MCP 服务器或 Agent 插件的工具函数中。

```python
from pathlib import Path
from typing import List

def is_path_allowed(target: str, allowed_dirs: List[str]) -> bool:
    """Return True if target path is inside one of the allowed directories."""
    # 1. 将输入转为 Path 对象并解析为绝对路径，同时展开符号链接
    resolved = Path(target).expanduser().resolve()

    # 2. 确保所有白名单目录也被同样解析
    for allowed in allowed_dirs:
        resolved_allowed = Path(allowed).expanduser().resolve()
        # 3. 检查是否在白名单目录或其子目录下
        try:
            resolved.relative_to(resolved_allowed)
            return True
        except ValueError:
            continue
    return False
```

使用示例：

```python
WHITELIST = [
    "/home/user/projects",
    "/var/log/myapp",
]

print(is_path_allowed("/home/user/projects/report.md", WHITELIST))  # True
print(is_path_allowed("~/projects/../.ssh/id_rsa", WHITELIST))      # False
print(is_path_allowed("/var/log/myapp/../other/file", WHITELIST))   # False (根据日志目录结构)
```

要点：

- `.expanduser()`：处理 `~` 开头的路径，避免 `~/allow/../.ssh` 被误判。
- `.resolve()`：同时完成**绝对路径化**与**符号链接解析**。这步是抵御路径穿越和软链接逃逸的关键。
- `.relative_to()`：如果目标路径不能以 `resolved_allowed` 为前缀（即不在其目录树下），会抛出 `ValueError`，此时继续检查下一个白名单项。

## 集成到 MCP / OpenClaw 插件

在 MCP 服务器中，可以在工具实现入口统一调用该函数。例如一个 `read_file` 工具：

```python
async def tool_read_file(path: str):
    if not is_path_allowed(path, config.allowed_directories):
        raise PermissionError(f"Access denied: {path}")
    return await read_file_content(path)
```

如果有多个工具需要文件访问，可以抽成中间件或装饰器。对于 Agent 工作流，建议早期就做检查，避免后续工具链在错误路径上继续操作。

## 踩坑点

1. **符号链接会破坏前缀假设**
   `Path.resolve()` 可以解决，但务必注意它也会跟随软链接并展开 `..` 分量。如果你的文件系统使用了 bind mount 或 union mount（如 Docker 卷、OverlayFS），解析后的路径可能与预期不同。建议测试环境中的实际挂载情况。

2. **Windows 下的盘符与长路径**
   虽然在 Agent 实践里多数部署在 Linux，但如果你的 MCP 服务器会跑在 Windows 上，要注意盘符大小写和 `\\?\` 前缀。`pathlib` 的 `resolve()` 在 Windows 上能处理大部分情况，但 `\\server\share` 类的 UNC 路径需要额外处理。可以统一要求 Windows 下白名单目录也是绝对路径并使用小写盘符比较。

3. **竞争条件 (TOCTOU)**
   检查通过后到实际文件操作之间，文件系统状态可能发生变化（如符号链接被替换）。在普通 Agent 场景中风险较低，但如果涉及高敏感数据和并发操作，可以参考 `openat` 的思路或检查后立即操作。一般工程实践可以接受这种残存风险。

4. **相对路径的解析基准**
   工具函数中需要确定“当前工作目录”。如果 Agent 或 MCP 服务器会切换 cwd，最好在解析之前使用绝对参数，或在调用 `is_path_allowed` 前 `path = os.path.abspath(path)`。在示例中 `Path(target).resolve()` 会以 cwd 为基准解析相对路径，因此保持进程 cwd 固定或显式转成绝对路径更安全。

## 可复用建议

- **将白名单检查封装成库**：可以参考上述核心逻辑，提供一个 `FileGuard` 类，初始化时传入白名单列表，后续调用 `guard.validate(path)`。这样不仅统一，还方便在单元测试中 mock。
- **白名单可配置化**：从环境变量或配置文件加载，避免硬编码。例如 `AGENT_ALLOWED_DIRS=/data/workspace,/tmp/agent`。
- **日志与审计**：每当拒绝访问时，记录被请求的路径、原始参数和当前时间。这些信息对排查误拒和攻击尝试很有用。
- **按读写权限细分**：部分场景可能希望白名单内再区分只读和可写。可以拓展为返回 `Literal["read", "write", "none"]`，进一步提升安全性。

## 总结

目录白名单是 Agent 文件系统访问的最低成本护栏，它不依赖复杂的权限系统，只需一个精心编写的路径验证函数。核心思想就是“先规范化，再比较前缀”，防范路径穿越和软链接逃逸。结合 MCP 工具层的统一检查，就能在保持自动化灵活性的同时，大幅降低意外文件操作带来的影响。

用几十行代码把边界锁住，是每一个 Agent 自动化实践者都应该迈出的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/4939388478f15cd5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/7d34bf8f02a1ec3f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/ab02b35c2e7d20eb.png)

