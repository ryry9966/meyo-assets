---
title: 给自动化脚本加本地目录白名单：Agent 文件访问护栏实战
feedId: 30905
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 Agent 开发中，我们经常让自动化脚本访问本地文件系统：读取配置、写入临时文件、导出结果，或者通过 MCP 工具调用外部数据源。问题在于，一旦脚本由模型生成或流程链路过长，路径拼接错误、未预期的符号链接、从日志中“学”到的绝对路径，都可能让进程触碰到它不该碰的目录。对于运行在本地但不可完全信任（如回放用户笔记、处理上传文件）的 Agent，这等同于一个定时炸弹。

多数生产级运行时（容器、Wasm sandbox）提供了文件系统隔离，但并非所有场景都方便立刻引入这些重依赖。一个轻量又可控的举措是：给脚本运行环境加一层**本地目录白名单**，强制所有文件 I/O 操作只能落在预先声明的目录集合内。本文介绍如何在 Python 自动化脚本中实现这样一个护栏，并分享从坑里爬出来的经验。

## 问题抽象

你有一个 Agent 内部脚本（或用函数封装的逻辑），它会在运行时读写本地文件。你希望对该脚本“可见”的文件系统仅限于：

- 项目工作区目录 `/app/workspace`
- 临时缓存目录 `/tmp/agent-cache`

除此以外的任何路径（如 `~/.ssh`、`/etc/passwd`）都应被拒绝访问，无论意图如何。

这需要从入口控制，而不是在每个 `open()` 调用处补 `if`。好的护栏应该是**透明且可审计**的。

## 做法与步骤

### 1. 规范化路径，白名单判定

核心思路：对待访问路径做 `realpath` 解析（消除符号链接、`..` 等），然后检查它是否以白名单中某一目录为前缀。用 Python 实现：

```python
from pathlib import Path
from typing import List

class FileAccessPolicy:
    def __init__(self, allowed_dirs: List[Path]):
        # 存储已解析的绝对路径
        self.allowed = [d.resolve(strict=True) for d in allowed_dirs]

    def is_allowed(self, target: Path) -> bool:
        try:
            real = target.resolve(strict=False)
        except (OSError, RuntimeError):
            return False
        # 必须严格处于某一目录之下，或等于该目录
        return any(real == allowed or allowed in real.parents for allowed in self.allowed)
```

- `resolve(strict=False)` 会把符号链接最终指向的真实路径解析出来，防止“~/safe_link -> /etc”这类绕过。
- 要求目录已存在时使用 `strict=True` 避免白名单配置笔误（若是动态创建的目录可延迟校验）。

### 2. 注入到脚本上下文

除了显式检查，更实用的方法是用一个**代理文件对象**或**自动包装函数**，把 `open`、`os.remove`、`shutil.move` 等全部接管。但为了轻量，可以只包装最常见的 `open`：

```python
import builtins

def guarded_open(file, mode='r', buffering=-1, encoding=None, *args, **kwargs):
    policy = get_current_policy()   # 从上下文变量获取
    path = Path(file).resolve()
    if not policy.is_allowed(path):
        raise PermissionError(f"Access denied for path: {file}")
    return builtins.open(file, mode, buffering, encoding, *args, **kwargs)
```

在 Agent 执行块内临时替换 `open`：

```python
import contextlib

@contextlib.contextmanager
def fs_guard(allowed_dirs):
    policy = FileAccessPolicy(allowed_dirs)
    token = set_current_policy(policy)    # contextvars 实现
    original_open = builtins.open
    builtins.open = guarded_open
    try:
        yield
    finally:
        builtins.open = original_open
        reset_current_policy(token)
```

然后：

```python
with fs_guard(["/app/workspace", "/tmp/agent-cache"]):
    run_agent_script()
```

这样一来，脚本里任何直接或间接的 `open()` 调用都会自动过护栏，不需改动原有业务代码。

### 3. 覆盖更多 I/O 入口（进阶）

如果脚本还通过 `pathlib.Path().write_text()` 或第三方库（如 `json.dump`）操作文件，需要额外挂钩。比如：

- Patch `Path.open`, `Path.write_text`, `Path.read_text`
- 对 `os.rename`, `os.remove`, `os.mkdir` 等函数也做替换（在 `os` 模块级别）

实践中，通常优先保护“读”和“写”操作。如果安全要求高，就上全套沙箱库（如 PyPy 的 sandbox，或用 `seccomp`），而不是纯 Python 白名单。

## 踩坑点

- **符号链接是绕过的头号通道**。仅做字符串前缀检查（如 `str(path).startswith(allowed)`) 非常脆弱。一定要 `resolve()` 后再判断。
- **`resolve()` 会抛出异常**：如果路径组件不存在或权限不够，调用可能失败。需要处理 `OSError`，可选择按保守策略“拒绝”。
- **Windows 盘符与大小写**：`C:\workspace` 和 `c:\workspace` 代表同一位置。`Path.resolve()` 会处理好，但自定义比较时要用 `os.path.normcase`。
- **临时文件与 /proc 特殊目录**：Linux 下写入 `/proc/self/fd/...` 也会产生文件访问，最好将 `/proc`, `/sys` 等特殊挂载点统一加入黑名单或白名单外。
- **多线程环境**：通过 `contextvars` 存储 `policy` 是线程安全的。如果脚本使用多进程，需在每个进程中重新设置策略。
- **白名单目录的 `resolve()` 时机**：如果目录尚不存在，`resolve(strict=True)` 会抛 FileNotFoundError。可以改为初始化时用 `parent.resolve()` 向上查找存在的部分来检查合法性，或者延迟到首次使用再校验。

## 可复用建议

- **配置外置**：通过环境变量 `SAFE_FS_DIRS` 传入逗号分隔目录列表，方便不同部署环境调整。
- **分层防御**：白名单仅是最后一道“铁网”，但不要依赖它防恶意代码。代理文件操作的同时应记录日志，方便审计。
- **配合 Agent 框架**：如果你用 OpenClaw 或其他框架运行插件，可以在工具调用前后包裹 `fs_guard` 上下文。甚至可以将 `FileAccessPolicy` 作为插件配置项，形成可共享的安全策略。
- **灰名单模式**：除了允许，还可以定义只读目录、只写目录。例如只允许读取 `/app/config`，不允许写入。
- **测试**：写一个单元测试，尝试访问白名单外的路径、通过符号链接绕过，确保策略生效。

## 总结

给 Agent 自动化脚本加本地目录白名单，是一个成本很低但回报明显的基础护栏。它不依赖外部沙箱，容易集成到现有的 Python 自动化流程中，能有效防止路径拼接错误、LLM 幻觉导致的不安全文件操作。核心就是三个动作：**路径规范化、白名单前缀判定、关键 I/O 函数替换**。踩过符号链接的坑后，会愈发明白：文件访问控制不能停留在字符串层面。最终，这套机制最适合作为多层防御中的一层，配合审计日志、最小权限账户、以及必要时引入的强隔离方案，让非完全受信的自动化脚本在受控范围内运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/cb000dad2012289f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/df2b039493c7dc19.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/be0ef035f45058e3.png)

