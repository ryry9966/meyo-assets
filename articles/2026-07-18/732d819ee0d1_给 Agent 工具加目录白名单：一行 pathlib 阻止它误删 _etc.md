---
title: 给 Agent 工具加目录白名单：一行 pathlib 阻止它误删 /etc
feedId: 29506
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

把本地文件操作能力交给 Agent（或 MCP 工具、插件自动化脚本）后，安全边界就从“人工审核命令”退化成“信任模型输入的路径”。如果工具实现只是简单拼接字符串 `os.remove(user_path)`，那么一个精心构造的 `../../.ssh/authorized_keys` 就可能在你不注意时被执行。本文以 Python 环境为例，讨论如何为这类自动化脚本加上一个**基于本地目录白名单的访问护栏**——不做容器化、不引入 SELinux，只靠代码里的路径验证。

适用场景：OpenClaw 自定义工具、MCP 服务器中的文件读写实现、自动化脚本封装成 function calling 的工具。

## 问题拆解

Agent 一般以如下形式暴露文件操作：

```python
def read_file(path: str) -> str:
    return open(path).read()
```

这个实现接受**任意绝对路径**，等于把整个文件系统开放给调用方。即便我们限定来自某个“工作区”，用户也可能用 `../` 突破边界。另外，符号链接、大小写不敏感文件系统（macOS/Windows）也会让简单的字符串前缀检查失效。

目标：**强制所有文件访问落在某个预定义的目录（及其子目录）内**，其余路径直接拒绝并返回明确错误。

## 实现步骤

### 1. 定义可信任的根目录

通过环境变量或配置文件注入，避免硬编码：

```python
import os
from pathlib import Path

SANDBOX_ROOT = Path(os.getenv("AGENT_WORKSPACE", "/tmp/agent_workspace")).resolve()
```

`.resolve()` 会消除符号链接、`..` 和多余的 `/`，保证是规范化绝对路径。

### 2. 设计路径校验函数

核心逻辑：把传入路径解析为绝对路径后，检查其是否位于 `SANDBOX_ROOT` 内。

```python
def validate_path(user_path: str, base: Path = SANDBOX_ROOT) -> Path:
    # 允许用户传相对路径，基于 base 拼接
    candidate = (base / user_path).resolve()
    # Python 3.9+ 的 is_relative_to 是最好的检查
    if not candidate.is_relative_to(base):
        raise PermissionError(f"Access denied: {candidate} is outside {base}")
    return candidate
```

如果要在旧版 Python（<3.9）中使用，可将 `is_relative_to` 替换为：

```python
base in candidate.parents or candidate == base
```

### 3. 在工具函数里嵌入检查

以读文件为例：

```python
def safe_read_file(path: str) -> str:
    safe_path = validate_path(path)
    if not safe_path.is_file():
        raise FileNotFoundError(f"Not a file: {safe_path}")
    return safe_path.read_text(encoding="utf-8")
```

同理，写文件、删除文件、列目录都必须先通过 `validate_path`。**列出目录的结果不要直接暴露绝对路径**，返回相对于 `SANDBOX_ROOT` 的路径更安全。

## 复合场景的踩坑点

### 1. 符号链接欺骗

假设工作区 `/workspace` 内有一个符号链接 `link -> /etc`，用户请求 `link/passwd`。  
如果我们仅仅做字符串拼接，会命中白名单，但 `.resolve()` 会还原为真实路径 `/etc/passwd`，从而被 `is_relative_to` 挡住。  
所以要**先拼接再 resolve**，不要上来 resolve 用户输入再手动拼接。

### 2. 不区分大小写的文件系统

macOS 默认文件系统不区分大小写，但 `Path` 对象保留原始大小写。`relative_to()` 比较的是路径对象，大小写不一致会导致抛出 `ValueError`。  
解决：统一用 `.resolve()` 得到系统实际大小写后的路径，再检查包含关系。

### 3. 并发/竞态条件

在 `validate_path` 和实际文件操作之间存在窗口，文件可能被替换为恶意符号链接（TOCTOU）。  
对于高风险操作（如执行脚本），建议直接使用 `open(file, 'r')` 前获取文件描述符并检查 `fstat` 是否为常规文件；或者在打开后立即检查路径是否仍符合白名单，但除非是安全敏感极高场景，否则更经济的方式是确保工作区目录权限正确，禁止非 root 用户创建符号链接到外部。

### 4. Windows 路径分隔符

如果要跨平台，注意 `pathlib` 处理 Windows 盘符，`is_relative_to` 在跨盘符时会直接 `False`，这符合预期。

## 可复用建议

- **抽象成装饰器**：为一系列工具函数提供统一的 `@require_sandbox` 装饰器，减少遗漏。
- **返回标准化错误**：对外抛出清晰的 `PermissionError`，Agent 可根据错误信息调整后续行为，避免幻觉重试。
- **结合 allowlist 与 denylist**：除了根目录白名单，还可以额外禁止某些敏感文件名（如 `.env`、`*.key`），这可以作为第二道防线。
- **MCP 工具实现**：如果使用 MCP 服务器，将 `validate_path` 放在工具 handler 的入口，并在返回错误时给出 `isError: true` 通知客户端。
- **日志记录**：记录所有被拒绝的访问请求，便于事后发现异常调用。

示例路径校验函数可复用：

```python
def require_sandbox(base_env="AGENT_WORKSPACE"):
    def decorator(func):
        base = Path(os.getenv(base_env, "/tmp/default_workspace")).resolve()
        def wrapper(user_path, *args, **kwargs):
            safe_path = (base / user_path).resolve()
            if not safe_path.is_relative_to(base):
                raise PermissionError(f"Blocked: {safe_path}")
            return func(str(safe_path), *args, **kwargs)
        return wrapper
    return decorator
```

## 总结

给自动化脚本增加本地目录白名单，核心就是“**基于规范化的真实路径，做归属检查**”。一行 `pathlib` 配合 `is_relative_to` 可以拦住绝大多数路径穿越。但它不能取代纵深防御：生产环境仍建议结合容器、只读挂载、专用服务账号等手段。把这条护栏作为工具实现的第一行代码，会比事后审计日志来得更省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/226bac1d992c6724.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/da7c19f5680e044b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/01a7583fbc407d29.png)

