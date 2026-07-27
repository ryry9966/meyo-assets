---
title: 给自动化脚本加一道本地目录白名单：Agent 文件访问护栏实践
feedId: 30689
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 OpenClaw 的 Agent 实践中，我们经常需要在自动化流程里执行用户提供的脚本——有时是 Python 片段，有时是完整的项目脚本。这些脚本往往需要读写文件：从指定目录读取输入，生成报告，或更新配置文件。但直接交给 Agent 任意文件系统访问权限，风险是明确的：一个不小心，它可能覆盖系统关键配置、删除数据目录，或者在无法预知的路径上制造副作用。

传统的容器隔离当然可靠，但很多场景下我们只想用轻量级方案，比如在同一个 Python 进程里运行一段自动化逻辑。这时候，主动“护栏”（guardrail）的价值就体现出来了：**只允许脚本访问预先设定的本地目录白名单，其他路径一律拒绝。**

## 问题拆解

假设我们有一个 MCP 工具或插件，它接收用户提交的脚本字符串和输入参数，在 Agent 中执行。需求很清晰：

- 提供一个 `allowed_dirs` 白名单，比如 `["/data/workspace", "/tmp/sandbox"]`。
- 脚本运行期间，所有文件 I/O（`open()`、`os.remove()`、`shutil.move()` 等）必须被限制在白名单内。
- 符号链接、相对路径、`../` 这种常见绕过手段需要被正确处理。
- 方案对现有脚本侵入性尽量小，最好不要给脚本作者增加额外的编码负担。

## 做法：上下文白名单 + 透明拦截

核心思路是利用 Python 的 `contextvars` 在请求级别存储白名单目录，同时通过运行时“接管”内置函数，在真正执行文件操作前做路径检查。

### 1. 定义白名单配置

白名单可以是列表，也可以是一个 YAML/JSON 配置。这里采用最简单的：在 MCP 工具接口层，调用者传入允许的路径。

### 2. 安全解析路径

所有文件路径在检查前都要：

- 使用 `os.path.abspath()` 转成绝对路径
- 使用 `os.path.realpath()` 解析符号链接和 `..`
- 确认最终的真实路径以白名单中某个目录开头（注意要处理目录分隔符，防止 `/data/workspace2` 被误认为 `/data/workspace` 的子目录）

```python
import os
from pathlib import Path

def is_path_allowed(path: str, allowed_dirs: list[str]) -> bool:
    # 规范化为绝对路径并解析符号链接
    real = os.path.realpath(os.path.abspath(path))
    for d in allowed_dirs:
        real_d = os.path.realpath(os.path.abspath(d))
        try:
            Path(real).relative_to(real_d)
            return True
        except ValueError:
            continue
    return False
```

### 3. 透明拦截文件操作

使用一个上下文管理器，通过 `sys.modules` 替换内置 `open`（以及 `os.open`、`os.remove` 等）为检查版本：

```python
import sys
import builtins
import contextvars

current_allowed_dirs = contextvars.ContextVar('allowed_dirs', default=None)

def _safe_open_wrapper(original_open):
    def safe_open(file, mode='r', *args, **kwargs):
        allowed = current_allowed_dirs.get()
        if allowed:
            # 创建/写入时检查目标路径；读取时也检查防止信息泄漏
            if not is_path_allowed(file, allowed):
                raise PermissionError(f"Access to {file} is not allowed")
        return original_open(file, mode, *args, **kwargs)
    return safe_open

class FileAccessGuard:
    def __init__(self, allowed_dirs):
        self.allowed_dirs = [os.path.realpath(d) for d in allowed_dirs]
        self._token = None
        self._original_open = builtins.open
        self._original_os_functions = {}

    def __enter__(self):
        # 设置 context var
        self._token = current_allowed_dirs.set(self.allowed_dirs)
        # 替换 open
        builtins.open = _safe_open_wrapper(self._original_open)
        # 替换 os 模块相关函数（os.remove, os.rename 等）
        import os as os_module
        for func_name in ('remove', 'rename', 'unlink', 'rmdir', 'mkdir'):
            original = getattr(os_module, func_name, None)
            if original:
                self._original_os_functions[func_name] = original
                def make_safe(orig=original):
                    def safe(*args):
                        allowed = current_allowed_dirs.get()
                        if allowed:
                            for a in args:
                                # 部分函数第一个参数是路径
                                if isinstance(a, (str, bytes, os.PathLike)):
                                    if not is_path_allowed(str(a), allowed):
                                        raise PermissionError(f"Operation on {a} is not allowed")
                        return orig(*args)
                    return safe
                setattr(os_module, func_name, make_safe(original))
        return self

    def __exit__(self, *args):
        # 恢复原状
        builtins.open = self._original_open
        import os as os_module
        for name, orig in self._original_os_functions.items():
            setattr(os_module, name, orig)
        current_allowed_dirs.reset(self._token)
```

在实际的 MCP 工具调用中，我们这样使用：

```python
def run_script_safe(script_code: str, allowed_dirs: list[str]):
    with FileAccessGuard(allowed_dirs):
        exec(script_code, {"__builtins__": __builtins__}, {})
```

> 这里为了演示简化了命名空间处理，真实工程中建议通过 `restricted_globals` 传入预定义的工具函数，避免脚本直接拿到完整 `__builtins__`。

### 4. 集成到 OpenClaw 插件

我们将 `FileAccessGuard` 集成到 MCP 工具或插件的一个动作中。每次 Agent 决定执行本地脚本时，都会携带一个 `allowed_dirs` 参数（可由平台管理员或用户预先配置）。这样即使用户提交的脚本有恶意倾向，越权操作也会被主动拦截并返回 `PermissionError`。

## 踩坑点

1. **符号链接绕过**  
   `os.path.realpath()` 可以解析符号链接，但如果白名单目录本身就是一个符号链接，需要提前用 `realpath` 处理，否则比较会失败。务必在初始化白名单和检查路径时都调用 `realpath`。

2. **相对路径与更改工作目录**  
   如果脚本里调用了 `os.chdir()`，相对路径的基础会变化。`is_path_allowed` 中使用的 `os.path.abspath(path)` 依赖当前工作目录，可能被绕过。解决办法：禁止 `os.chdir`（同样拦截），或在检查时强制基于初始工作目录解析。最简单的是将 `chdir` 加入拦截列表。

3. **低层次文件描述符操作**  
   `os.open()` 返回 fd，之后 `os.fdopen()` 会构造文件对象。必须同时拦截 `os.open` 并检查路径。此外，一些第三方库可能使用 `openat()` 或直接 `syscall`，这些 hook 不到，但对大多数 Python 自动化脚本而言，拦截 `open` 和一系列常用 `os` 函数已经足够。

4. **多线程环境**  
   `contextvars` 天然支持协程和线程，每个线程会得到自己的副本，不会互相干扰。但如果脚本内部使用了多线程，`current_allowed_dirs.get()` 依然能正常传递，因为子线程默认继承父线程的 context。

5. **性能**  
   每次文件操作都要解析真实路径，高频场景下会有开销。可以引入一个带 TTL 的路径缓存（`functools.lru_cache`），但要小心白名单动态变化的情况。在护栏层面，安全优先于性能，这个开销通常可以接受。

## 可复用建议

- **封装为独立包**：把 `FileAccessGuard` 提炼成一个 Python 包，支持通过环境变量、配置文件或参数指定白名单，并提供 `@guard_required` 装饰器。
- **与 MCP 工具深度绑定**：在 OpenClaw 的 MCP 工具定义中，将 `allowed_dirs` 作为必选配置，强制由平台注入，避免脚本作者自行指定。
- **日志与审计**：所有被拒绝的操作记录到日志，Agent 运行时可以看到清晰的失败原因，便于排障。
- **渐进放宽**：对于信任的脚本，可以设置 `allowed_dirs=[""]` 以禁用检查（但保留拦截框架，方便随时收紧）。
- **结合只读挂载**：如果环境允许，将敏感的宿主目录挂载为只读或不可见，作为纵深防御。

## 总结

给自动化脚本加上本地目录白名单并不复杂，本质上就是把文件操作路径全部转为真实路径，再与预设的安全集合做前缀匹配。通过 `contextvars` 和运行时内置函数替换，我们可以在不修改用户脚本的前提下，让 Agent 获得一个“软沙箱”。

在 OpenClaw 这样的 MCP 工具实践中，这种护栏可以为插件作者和平台运营者提供最低限度的文件安全保障，避免 Agent 从“自动执行”变成“自动灾难”。建议大家在需要执行用户上传脚本的场景中，至少实现这样一层轻量级文件访问控制。

---

