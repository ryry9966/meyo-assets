---
title: 给 Agent 自动化脚本加上本地目录白名单：文件访问护栏实战
feedId: 30922
source: 综合讨论
publishedAt: 2026-07-29
---

# Agent 文件访问护栏：给自动化脚本加本地目录白名单

## 背景
在 OpenClaw 生态里，越来越多的用户通过 MCP 服务或自研插件让 Agent 直接操作本地文件系统——比如让脚本读写数据、清理临时目录、解析日志等。问题在于：一旦自动化脚本逻辑出错、或 prompt 被恶意注入，Agent 就可能越权读取敏感配置文件、删除系统文件，甚至把整个用户目录打包送走。

把 Agent 放在沙箱里运行是一种办法，但很多时候沙箱太重，开发和调试也不方便。更轻量的做法是在文件 I/O 层面加一道**本地目录白名单护栏**：只允许脚本在指定的目录（例如项目 workspace）内进行读写，其他路径一律拒绝。这样即使行为失控，影响也被严格限定在一个小范围内。

## 问题定义
为一个 Python 自动化脚本增加白名单控制，要求：
- 仅允许访问预先配置的几个目录（及其子目录）
- 拦截 `open()`、`os.remove()`、`shutil.copy()` 等常见文件操作
- 防止符号链接、路径拼接等手法绕过
- 实现尽量非侵入，便于集成到现有 Agent 工具调用中

## 实现步骤

### 1. 设计白名单校验核心
使用 `pathlib.Path.resolve()` 将用户传入的路径转换为**不含符号链接的绝对路径**，然后判断其是否以任一白名单目录为前缀。这样能避免 `symlink` 和 `../` 跳过。

```python
from pathlib import Path

class PathGuard:
    def __init__(self, allowed_dirs: list[str]):
        self.allowed = [Path(d).resolve() for d in allowed_dirs]

    def is_allowed(self, target: str | Path) -> bool:
        try:
            real_path = Path(target).resolve()
        except (OSError, RuntimeError):
            return False
        return any(
            real_path.is_relative_to(d) for d in self.allowed
        )
```

`is_relative_to()` 是 Python 3.9+ 提供的方法，能准确判断路径是否在某个目录树内。

### 2. 包装文件操作
对常用的文件读、写、删除、复制进行包装，内部先做白名单校验，再执行原生操作。

```python
import shutil
from typing import Optional

class SafeFS:
    def __init__(self, guard: PathGuard, cwd: Optional[Path] = None):
        self.guard = guard
        # 非 None 时，相对路径将基于此工作目录解析
        self.cwd = Path(cwd).resolve() if cwd else None

    def _resolve(self, path: str | Path) -> Path:
        p = Path(path)
        if not p.is_absolute() and self.cwd:
            p = self.cwd / p
        return p.resolve()

    def open(self, path, mode='r', *args, **kwargs):
        real = self._resolve(path)
        if not self.guard.is_allowed(real):
            raise PermissionError(f"Access denied: {real}")
        return open(real, mode, *args, **kwargs)

    def remove(self, path):
        real = self._resolve(path)
        if not self.guard.is_allowed(real):
            raise PermissionError(f"Access denied: {real}")
        # 建议额外判断不要删除目录本身
        os.remove(real)

    def copy(self, src, dst):
        # src 和 dst 都必须通过白名单
        ...
```

当 Agent 调用工具函数时，只需要改用 `safe_fs.open()` 代替原生 `open()`，或通过上下文管理器注入。

### 3. 集成到 Agent 工具
假设你的 Agent 通过 MCP 暴露了一个 `read_file` 工具，实现可以这样改写：

```python
safe_fs = SafeFS(PathGuard(["/home/user/myproject/workspace"]),
                 cwd="/home/user/myproject/workspace")

def mcp_read_file(path: str) -> str:
    with safe_fs.open(path, 'r') as f:
        return f.read()
```

这样无论 Agent 传入什么路径，都无法超出 workspace。

## 踩坑记录

### 相对路径与工作目录
如果 Agent 脚本的当前工作目录不在白名单内，而你允许相对路径解析，就必须让 `SafeFS` 强制绑定一个安全的工作目录（`cwd` 参数），否则相对路径会基于真正的工作目录解析，可能指向 `/etc`。我在首次实现时漏掉了这个绑定，导致相对路径 `../../etc/passwd` 成功绕过了校验。

### 符号链接的死穴
即便你检查了路径前缀，如果白名单内存在一个指向 `/etc` 的符号链接，那么通过该链接仍可访问外部文件。`Path.resolve()` 会跟随符号链接，因此最终真实路径不会以白名单开头，遂被拒绝。但务必须每次都调用 `resolve()`，避免为了性能使用 `real_path` 缓存而遗漏。

### 目录删除保护
`os.remove()` 可以删除文件，但 `shutil.rmtree()` 能直接删除整个目录。如果在白名单根目录（如 workspace）上调用 `rmtree`，将清空整个工作区。生产环境中，建议在 `remove` 与 `rmtree` 上加一层判断，禁止删除白名单的顶层目录本身，或者要求用户显式传入一个 force 参数。

### 跨平台路径陷阱
Windows 下盘符不区分大小写但 `is_relative_to` 区分大小写，可以通过先将 `allowed` 和 `real_path` 统一转换为小写（或使用 `os.path.normcase`）来避免。另外 UNC 路径、`\\?\` 前缀也需要提前规范化。

## 可复用建议
- **配置化白名单**：通过环境变量 `SAFE_FS_ALLOWED_DIRS` 传入逗号分隔的目录列表，启动时加载。
- **Context Manager**：提供 `with SafeFS.session(allowed_dirs)` 上下文，内部 monkey-patch `os.open` 等操作（生产环境慎用，适合测试）。
- **MCP 工具层拦截**：如果使用 OpenClaw 的插件系统，可以写一个装饰器自动对带有 `path` 参数的函数进行白名单检查，让开发者无需在每个工具里重复代码。
- **审计日志**：在拒绝访问时记录日志，包含时间、原始路径、解析后路径、调用栈，方便排查 Agent 的异常行为。

## 总结
本地目录白名单是 Agent 文件操作的第一道防线，实现成本极低，却能有效防止灾难性误操作。工程上只需要几十行代码，配合路径规范化、符号链接解析和工作目录绑定，就能把失控脚本限制在安全区域内。在正式上线前，不妨先把这道护栏加上——你可能永远用不上，但一旦需要，它会让你避免加班删库跑路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/42aa4706c8bfcb11.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/16339d49815f2137.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/8281d6818e5e0cc2.png)

