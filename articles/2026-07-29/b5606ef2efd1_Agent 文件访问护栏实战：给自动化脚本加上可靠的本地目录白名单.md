---
title: Agent 文件访问护栏实战：给自动化脚本加上可靠的本地目录白名单
feedId: 30936
source: 综合讨论
publishedAt: 2026-07-29
---

# Agent 文件访问护栏实战：给自动化脚本加上可靠的本地目录白名单

## 背景

在 OpenClaw 等 Agent 框架中，我们经常通过自定义工具（Tool）或 MCP 服务器为 Agent 开放文件读写能力。例如，让 Agent 帮你整理下载目录、生成代码并写入工程目录、或者分析日志文件。这些操作一旦放开，Agent 就获得了几乎等同当前用户权限的文件系统访问。仅靠 Prompt 约束是不可靠的——模型可能被误导、幻觉产生危险路径，或被间接注入要求删除配置文件。因此，我们需要在工程侧实现一套可执行的**文件访问护栏**，把操作限制在预定义的本地目录白名单内。

## 问题拆解

常见风险包括：
- **路径穿越**：`../../etc/passwd` 直接突破预期工作目录。
- **符号链接绕过**：通过软链接指向敏感位置，绕开前缀检查。
- **绝对路径误用**：Agent 被要求输出 `/home/user/.ssh` 等。
- **竞态条件**：在检查与操作之间路径被替换（尤其在可写目录下）。

我们的目标是为所有 Agent 发起的文件操作加上一层白名单校验，且该校验必须解析到真实路径（realpath），并在每次 I/O 前都执行，不能信任前一次的缓存。

## 核心实现：SecureFileAccess 类

下面给出一个可直接复用的 Python 实现，使用 `os.path.realpath` 解析符号链接和相对路径，再与白名单前缀进行匹配。为处理新建文件场景（路径尚未存在），对不存在路径退回校验其父目录。

```python
import os
from pathlib import Path
from typing import List, Optional

class FileAccessGuard:
    """限制所有文件操作必须在白名单目录内。"""

    def __init__(self, allowed_dirs: List[str]) -> None:
        # 将白名单目录统一解析为绝对真实路径
        self._allowed = [os.path.realpath(d) for d in allowed_dirs]

    def _resolve(self, path: str) -> Optional[str]:
        """安全解析路径，失败返回 None。"""
        try:
            # 若路径存在直接解析，否则尝试解析其父目录
            if os.path.lexists(path):
                return os.path.realpath(path)
            parent = os.path.dirname(path)
            if not parent:
                return None
            real_parent = os.path.realpath(parent)
            return os.path.join(real_parent, os.path.basename(path))
        except (OSError, ValueError):
            return None

    def is_allowed(self, path: str) -> bool:
        real = self._resolve(path)
        if real is None:
            return False
        # 必须属于某个白名单前缀，防止前缀部分匹配（如 /var/app 不能匹配到 /var/app-secret）
        for allowed in self._allowed:
            if real == allowed or real.startswith(allowed + os.sep):
                return True
        return False

    def read_text(self, path: str) -> str:
        if not self.is_allowed(path):
            raise PermissionError(f"File access denied: {path}")
        with open(path, "r", encoding="utf-8") as f:
            return f.read()

    def write_text(self, path: str, content: str) -> None:
        if not self.is_allowed(path):
            raise PermissionError(f"File access denied: {path}")
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, "w", encoding="utf-8") as f:
            f.write(content)

    def list_dir(self, path: str) -> List[str]:
        if not self.is_allowed(path):
            raise PermissionError(f"Directory access denied: {path}")
        return os.listdir(path)
```

在 OpenClaw 的 Tool 或 MCP 工具函数中，只需要实例化一个全局 `guard = FileAccessGuard(allowed_dirs=["/home/user/project", "/tmp/agent-workspace"])`，然后在每个工具函数入口调用 `guard.is_allowed()` 即可实现拦截。

## 集成到 Agent 工作流

以 OpenClaw 自定义 Tool 为例：

```python
from openclaw.tools import tool

guard = FileAccessGuard([
    os.path.expanduser("~/agent-sandbox"),
    "/var/tmp/agent-io"
])

@tool
def read_file(file_path: str) -> str:
    """读取文件内容，仅限安全目录。"""
    return guard.read_text(file_path)

@tool
def write_file(file_path: str, content: str) -> str:
    """写入文件，仅限安全目录。"""
    guard.write_text(file_path, content)
    return f"Written to {file_path}"
```

如果团队使用 MCP 服务器提供文件系统能力，可在 `mcp-server-filesystem` 等开源实现中加入此 Guard，或封装一层代理。无论哪种形态，护栏逻辑都应该集中管理，避免散落在多个工具中。

## 踩坑记录

1. **`os.path.realpath` 可能抛出异常**  
   解析不存在的路径时，部分平台会抛出 `FileNotFoundError`，所以采用 `lexists` 判断并回退到父目录拼接。注意 `realpath` 本身可能因权限问题失败，此时直接拒绝访问，按最安全策略处理。

2. **Windows 盘符与大小写**  
   在 Windows 上，`os.path.realpath` 会添加盘符和反斜杠，需要统一 `os.sep` 处理。白名单路径最好也用 `os.path.realpath` 预处理，并全部小写化（`os.path.normcase`）以应对大小写不敏感。

3. **目录结尾斜杠陷阱**  
   前缀检查必须加上 `os.sep`，否则 `/var/app` 可能匹配到 `/var/app-secret`。代码中使用 `real.startswith(allowed + os.sep)` 并且额外判断相等的情况。

4. **符号链接循环**  
   `realpath` 会检测符号链接循环，但极个别文件系统（如某些网络挂载）可能不抛出异常而返回错误值，需要超时或 fallback。

5. **新建文件的路径穿越**  
   当写入一个不经父目录校验的路径时，虽父目录在白名单，但文件名本身如 `../../secret` 可能组合出越权。因此 `_resolve` 必须解析组合后的完整路径，而不是仅校验父目录。

6. **并发环境下的 TOCTOU**  
   文件系统检查和使用存在微小窗口，但目前没有完美内核级解决方法。在可接受范围内，可通过禁止 Agent 操控可写目录的权限来缓解，或使用 Linux 的 `openat2` 与 `RESOLVE_NO_SYMLINKS` 等特性，但需要更底层实现。

## 可复用建议

- **环境变量配置**：将白名单路径通过 `AGENT_ALLOWED_DIRS` 环境变量注入，多个路径用冒号分隔，方便在不同部署环境中切换。
- **装饰器封装**：编写 `@restrict_paths` 装饰器，自动从工具参数中提取路径参数进行校验，减少重复代码。
- **日志与审计**：每次拒绝访问时记录详细日志（路径、时间、调用栈），帮助发现 Agent 的异常行为并优化 Prompt。
- **测试用例**：为安全层编写专用测试，覆盖符号链接、相对路径、不存在路径、大小写变体等，避免回归。
- **与容器配合**：在 Docker 容器中使用时，可将白名单目录挂载为只读或 `tmpfs`，双重防护。但即便有容器，进程内的 Guard 依然有用，防止 Agent 在同一容器内触碰敏感文件。

## 总结

给 Agent 脚本加文件目录白名单并不是新概念，却是工程化 Agent 必备的安全底线。本文提供的 `FileAccessGuard` 实现轻量、无外部依赖，可以直接嵌入现有 Tool 或 MCP 服务器中。核心原则是：**永远不要相信字符串路径，始终解析真实路径后做前缀匹配，并在每次 I/O 操作前重新检查**。配合合理的白名单配置和日志审计，我们能放心地让 Agent 操作文件系统，而不会担心意外删除系统配置或泄露敏感数据。

让护栏成为自动化脚本的默认配置，而不是事后的补丁。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/7a0c5a148d68453a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/c815d5830eda7e88.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/ff1e1cd73cffcbb5.png)

