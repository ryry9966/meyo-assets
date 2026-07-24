---
title: 为 Agent 工具注入安全护栏：本地目录白名单的工程化实现
feedId: 30284
source: 综合讨论
publishedAt: 2026-07-24
---

## 1. 背景：Agent 的“脚”需要被看见

在 OpenClaw、MCP 插件或任何允许 Agent 通过脚本访问本地文件系统的自动化场景中，文件读写几乎是最基础的能力。数据分析、报告生成、缓存操作、日志落盘——都离不开 `open()` 或 `os.remove()`。但一旦把文件系统暴露给由 prompt 驱动的执行链，风险就来了：

- 误操作：Agent 可能删除项目外的重要文件。
- 数据泄露：无意的 `cat ~/.ssh/id_rsa`。
- 供应链陷阱：恶意提示词诱导访问 `/etc/passwd` 或 `.env`。

在生产环境的 Agent 实践中，“信任但要限制”是基本原则。给脚本套上一个**本地目录白名单护栏**，即只允许访问预先定义的若干安全目录，是最低成本的防御措施之一。本文将给出一个轻量、可验证、可复用的工程化方案。

## 2. 问题定义

一个典型场景：你写了一个 Python 工具函数，作为 MCP 工具暴露给 OpenClaw Agent，用来读取用户提供路径的文件内容。如果不加限制，Agent 可以传入 `../../secret.txt` 从而突破当前工作目录。我们需要一个组件，保证任何被 Agent 调用的文件操作，只能发生在指定的目录下（例如 `./workspace/`, `/data/sandbox/`），其余路径一律拒绝。

要求：
- 对符号链接、相对路径、路径穿越具备防御能力。
- 对大小写敏感文件系统、Windows 长路径等差异有应对（至少不因平台导致绕过）。
- 接入成本低，不侵入既有业务代码，适合作为装饰器或上下文管理器复用。

## 3. 实现路径

### 3.1 核心函数：路径合法性校验

用 Python 的 `pathlib` 可以覆盖大部分平台差异，关键点是 **resolve()** ——它会消除 `..`、符号链接，并返回绝对路径。

```python
from pathlib import Path

class SafePathChecker:
    def __init__(self, allowed_dirs: list[Path]):
        # 将白名单目录都预先 resolve，消除环境差异
        self.allowed = [d.resolve() for d in allowed_dirs]

    def check(self, target: Path) -> Path:
        resolved = target.resolve()
        for allowed in self.allowed:
            # 使用 Path.is_relative_to（3.9+）或 commonpath 模仿
            try:
                resolved.relative_to(allowed)
                return resolved
            except ValueError:
                continue
        raise PermissionError(f"Access denied: {target}")
```

- `Path.relative_to()` 在 Python 3.9 以上可用，如果目标是 `allowed` 的子路径则不会抛出异常。
- 对 Windows 驱动器盘符，`resolve()` 会补全为 `C:\...`，白名单同样需要绝对路径。

### 3.2 接入 Agent 工具

假设你有一个文件读取工具，供 Agent 调用：

```python
def read_file(file_path: str):
    with open(file_path, 'r') as f:
        return f.read()
```

引入护栏后，改为：

```python
checker = SafePathChecker([
    Path("./sandbox/workspace").resolve(),
    Path("/tmp/agent_cache").resolve(),
])

def read_file_guarded(file_path: str):
    safe_path = checker.check(Path(file_path))
    with open(safe_path, 'r') as f:
        return f.read()
```

若传入 `../../.env`，`check()` 会抛出 `PermissionError`，工具调用直接失败。这种失败可以被 Agent 框架捕获并反馈给模型，提示“文件不在允许范围内”。

### 3.3 装饰器 & 上下文管理器封装

为了复用，建议封装为装饰器，自动校验所有参数中类型为路径的输入：

```python
from functools import wraps

def guard_paths(*param_names):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 简单实现：从 kwargs 中取出需要校验的路径参数
            for name in param_names:
                if name in kwargs:
                    kwargs[name] = checker.check(Path(kwargs[name]))
            return func(*args, **kwargs)
        return wrapper
    return decorator

@guard_paths('input_path', 'output_path')
def process_csv(input_path, output_path):
    # 此时两个路径已经被解析并校验
    ...
```

上下文管理器也很有用，适用于需要临时创建文件的操作：

```python
from contextlib import contextmanager

@contextmanager
def safe_open(path, mode='r'):
    p = checker.check(Path(path))
    f = open(p, mode)
    try:
        yield f
    finally:
        f.close()
```

## 4. 踩坑与补救

- **符号链接穿透**：攻击者可能在白名单目录内创建指向 `/etc/passwd` 的符号链接。`Path.resolve()` 会跟随链接并解析为真实路径，所以必须使用 `resolve()` 而非 `absolute()`。  
- **大小写与 NTFS**：Windows 下 `Path.resolve()` 会把路径转为系统实际大小写形式，但白名单目录也必须是相同形式，建议统一用小写或预先 `resolve()`。  
- **相对路径基准问题**：`Path(file_path)` 的相对基准是当前工作目录。如果 Agent 可能在运行中改变 `os.chdir()`，建议在 `check()` 内部强制转为绝对路径（`Path(file_path).resolve()` 已包含）。  
- **白名单配置来源**：不要硬编码到源码，可从环境变量 `AGENT_ALLOWED_DIRS` 读取，冒号分隔。但要防止此环境变量被 Agent 自身修改，通常预先在启动脚本中注入。  
- **并发场景**：`SafePathChecker` 实例无状态，线程安全。每次 `resolve()` 是纯操作，无副作用。  
- **异常处理**：`PermissionError` 可以被上层捕获，但建议直接让它上浮，由 Agent 框架当作工具调用错误自然终止，避免吞异常导致模型误以为操作成功。

## 5. 可复用建议

- **独立模块**：将 `SafePathChecker` 与装饰器抽离成一个 `file_guard.py`，在多个 Agent 工具间共享。依赖仅标准库 `pathlib`。  
- **测试套件**：准备若干恶意路径用例（`../etc/passwd`, `file:///etc/passwd`, 符号链接，空路径，绝对路径在外，`/etc/passwd` 等），确保全部被拒绝。  
- **与 MCP 集成**：若你用的是 MCP 服务器，可将这个检查器放在工具处理函数的最外层，避免重复实现。  
- **而非 OS 级沙箱替代**：该护栏是应用层防御，无法替代 Docker、seccomp 等 OS 级隔离。两者结合更稳固。

## 6. 总结

给 Agent 加上文件访问目录白名单，就像给自动化脚本戴上“隐形狗链”——它仍然灵活，但不能越界。实现仅需数十行代码，却能挡住大部分因 prompt 不可控引发的路径穿越风险。

在生产化 Agent 的过程中，这类工程护栏不是可选项，而是开跑前的必备检查项。建议在项目初期就落地，并以测试用例固化边界，让每一次文件读写都经过明确的授权校验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/352efb421e8047ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/86f4258f3c5c3e50.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/06bfaac5e5f52c4e.png)

