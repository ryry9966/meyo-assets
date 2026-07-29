---
title: 给 Agent 脚本加本地目录白名单：一个可落地的文件访问护栏方案
feedId: 30892
source: 综合讨论
publishedAt: 2026-07-29
---

# 给 Agent 脚本加本地目录白名单：一个可落地的文件访问护栏方案

## 背景

当我们在 OpenClaw 这类智能体框架中接入本地工具（如文件读写、执行脚本、读写配置），Agent 不再只是一个对话模型，而是一个拥有真实文件系统操作能力的执行单元。如果没有访问边界，一次 Prompt 注入或逻辑错误就可能导致关键文件被覆盖、隐私数据泄露甚至系统损坏。为此，我们需要一个**文件访问护栏**，只允许 Agent 在预先批准的目录内读写。本方案用**本地目录白名单 + 路径规范化验证**实现最小权限的文件访问控制。

## 问题定义

典型的 Agent 工具定义如下（伪代码）：

```python
def read_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()
```

如果 `path` 是 `/etc/passwd` 或 `~/.ssh/id_rsa`，Agent 将直接读取系统敏感文件。即使你开始只给它一个工作目录 `workspace/`，攻击者也可能通过 `../`、符号链接或绝对路径逃逸。

因此，护栏的核心任务是：**在接受任何文件路径参数并执行实际 IO 前，确保规范化后的路径位于安全目录集合内**。

## 实现步骤

### 1. 定义白名单目录集合

使用绝对路径列表，通常包括：

- 项目工作区：`/home/user/agent_workspace`
- 临时文件目录：`/tmp/agent_tmp`
- 只读资源目录：`/opt/agent_resources`

白名单必须以绝对路径形式定义，避免相对路径混淆。

### 2. 实现路径安全校验函数

核心逻辑：使用 `os.path.realpath()` 解析所有符号链接与相对路径，然后检查结果是否以任一白名单目录为前缀。

```python
import os
from typing import List

class FileAccessGuard:
    def __init__(self, allowed_dirs: List[str]):
        # 预处理白名单，统一规范化并去除末尾斜杠
        self.allowed_dirs = [os.path.realpath(d).rstrip(os.sep) for d in allowed_dirs]

    def check(self, path: str) -> str:
        # 解析绝对路径，消除符号链接和 ../
        real_path = os.path.realpath(path)
        # 检查是否以任意白名单目录开头
        for allowed in self.allowed_dirs:
            if real_path.startswith(allowed + os.sep) or real_path == allowed:
                return real_path
        raise PermissionError(f"Access denied: {path} -> {real_path}")
```

注意：必须同时检查 `real_path == allowed`，因为直接访问白名单目录本身也应被允许（例如列出目录内容）。

### 3. 集成到工具调用点

在文件 IO 工具的实现中，所有路径参数先经过 `guard.check()` 再使用：

```python
guard = FileAccessGuard(["/app/workspace", "/app/output"])

def safe_read_file(path: str) -> str:
    safe_path = guard.check(path)
    with open(safe_path, 'r') as f:
        return f.read()

def safe_write_file(path: str, content: str) -> None:
    safe_path = guard.check(path)
    with open(safe_path, 'w') as f:
        f.write(content)
```

对于 OpenClaw 的 MCP 插件，可以在 `execute_tool` 装饰器中加入此校验，实现统一拦截。

### 4. 相对路径与工作目录的处理

如果 Agent 工具允许传入相对路径，必须在校验前与当前工作目录（CWD）拼接并规范化。建议在工具侧明确 CWD 并也将其加入白名单，或强制要求工具调用只能使用绝对路径。

## 踩坑点

- **符号链接绕过**：攻击者可能在工作区内创建指向白名单外目录的软链接。`realpath()` 能完全解析，但需注意中间目录更新（如替换链接）可能导致 TOCTOU 问题。在高安全场景下，建议在文件操作前再次校验，或直接禁用工作区内的符号链接创建权限。
- **路径分隔符与尾随斜杠**：Windows 下盘符和反斜杠可能造成 `startswith` 误判。务必统一使用 `os.sep` 和 `os.path.realpath` 并处理大小写敏感问题（Windows 下 `realpath` 也会统一大小写）。
- **空白名单**：如果白名单未包含任何目录，所有访问都会被拒绝。启动时需明确日志输出白名单列表，避免配置遗漏导致 Agent 功能全部失败。
- **性能**：每次文件操作都调用 `realpath` 可能在大量小文件读写时引入开销。可结合缓存（LRU）优化，但需注意文件系统变化时的缓存失效。
- **临时文件目录陷阱**：若白名单包含 `/tmp`，其他进程也可能在此目录写入，需结合更细粒度的子目录隔离，如创建 Agent 专属的 `/tmp/agent-<uuid>` 子目录并加入白名单。

## 可复用建议

1. **封装成独立模块**：将 `FileAccessGuard` 设计为无状态的校验类，在 Agent 启动时初始化，并作为依赖注入到各个文件工具中。
2. **拒绝访问时记录审计日志**：记录原始请求路径、规范后路径、时间戳、调用栈，便于追踪异常行为。
3. **提供配置化接口**：通过环境变量或配置文件指定白名单，支持动态更新（例如通过 Admin API 重新加载，但需严格鉴权）。
4. **测试用例覆盖**：针对常见绕过编写单元测试，如 `../etc/passwd`、符号链接、Windows 绝对路径、长路径等。
5. **结合只读标记**：对资源目录设置只读权限，在工具调用前检查操作类型与目录权限是否匹配，实现更细粒度控制。

## 总结

文件访问护栏是 Agent 安全基线的核心组件。通过本地目录白名单 + `os.path.realpath` 路径规范化，我们能以较低成本实现边界隔离。该方案适用于 OpenClaw 插件开发、自动化脚本封装以及任何需要给本地文件操作加保险的场景。请务必将白名单检查放在最靠近系统调用的位置，并定期审计你的目录权限与符号链接风险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/9ab9674d4e962848.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/405d251e151a0db8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/e14e7bff8ffa2f87.png)

