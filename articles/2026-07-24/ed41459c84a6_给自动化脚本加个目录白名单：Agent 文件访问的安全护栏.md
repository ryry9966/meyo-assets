---
title: 给自动化脚本加个目录白名单：Agent 文件访问的安全护栏
feedId: 30269
source: 综合讨论
publishedAt: 2026-07-24
---

# 给自动化脚本加个目录白名单：Agent 文件访问的安全护栏

## 背景：一次“小脚本”引发的冷汗

几个月前我用 OpenClaw 搭了个自动化管线：Agent 从邮箱抓取附件，交给下游脚本处理并归档。有一天它收到一封带同名压缩包的邮件，解压后覆盖了当前目录下一个同名但完全无关的配置文件，导致链路中断近两个小时。

不是代码逻辑错，也不是权限溢出——仅仅是**自动化脚本缺少对“该碰哪些目录”的显式约束**。

大多数 Agent 或自动机码农倾向于给足权限、快速跑通。但当脚本开始读写文件系统，而没有限制可访问的范围，就容易出现：
- 误删/覆盖关键配置或数据；
- 被恶意构造的路径（如 `../../../etc/passwd`）诱使越权操作；
- 临时文件污染工作区外目录。

为此我实现了一个轻量的**文件系统护栏层**：只允许 Agent 在预设的目录白名单内操作，任何越界调用都在代码层面被拦截。

---

## 问题定义：我们要约束什么？

在 Agent/插件/工具函数中，常见的文件操作包括：
- `open()` 读写文件
- `os.remove()`, `shutil.rmtree()` 等删除操作
- `os.rename()`, `shutil.move()` 等移动操作
- `Path.read_text()`, `Path.write_bytes()` 等路径对象方法

护栏的目标：对**所有文件 I/O 调用**进行路径检查，确保其解析后的绝对路径以白名单中的某个目录为前缀。拒绝任何不在白名单内的访问，并记录告警。

---

## 实践步骤：一层透明的“路径锁”

以下基于 Python 实现，但思路适用于任何语言。核心是**统一所有路径操作入口**，注入检查逻辑。

### 1. 定义白名单与安全解析函数

```python
import os
from pathlib import Path

WHITELIST = [
    "/data/agent_workspace",
    "/tmp/agent_sandbox"
]

def is_allowed(path: str | Path) -> bool:
    """检查路径是否在白名单内，同时防御符号链接绕过"""
    # 解析为绝对路径，并消除符号链接
    real_path = Path(path).resolve(strict=False)
    for allowed in WHITELIST:
        allowed_real = Path(allowed).resolve(strict=True)
        # 必须判断共同前缀而非简单 startswith，防止 /data/agent_workspace 与 /data/agent_workspace_evil 混淆
        try:
            real_path.relative_to(allowed_real)
            return True
        except ValueError:
            continue
    return False
```

这里 `resolve(strict=False)` 会把 `..`、`.` 和符号链接全部化解为绝对路径，再使用 `relative_to` 做严格前缀匹配。  
（注：`strict=False` 在 Python 3.6+ 可用，用于路径不存在时也不抛异常。）

### 2. 包装文件操作

不要在每个调用点手动检查，而是用**安全的代理函数**替换内建的文件操作。

```python
import builtins
import shutil
import logging

original_open = builtins.open

def safe_open(file, mode='r', *args, **kwargs):
    if is_allowed(file):
        return original_open(file, mode, *args, **kwargs)
    else:
        logging.error(f"Blocked open: {file}")
        raise PermissionError(f"Access denied: {file}")

# 同理包装 remove, rmtree, rename 等
def safe_remove(path):
    if is_allowed(path):
        os.remove(path)
    else:
        logging.error(f"Blocked remove: {path}")
        raise PermissionError(f"Access denied: {path}")
```

**侵入性最低的方案**：在 Agent 脚本入口处替换命名空间中的这些函数名，或使用自定义的 `open = safe_open`，让后续代码无感知。如果代码使用 `pathlib.Path`，可以通过子类化覆盖 `open`、`unlink` 等方法，但成本稍高。

另一种方式：用上下文管理器临时覆盖 `builtins.open` 等，缺点是会影响所有 import，不推荐。

### 3. 在 Agent 或插件中集成

对于 OpenClaw 的插件或工具函数，可以把安全函数注入到 `globals()` 或通过基类工具，确保文件操作都通过护栏。  
比如在自定义的工具基类中：

```python
class SandboxedTool:
    def open(self, *args, **kwargs):
        return safe_open(*args, **kwargs)
    def remove(self, path):
        safe_remove(path)
```

传给 Agent 的 callable 对象只暴露 `SandboxedTool` 的方法，而不是原生函数。

---

## 踩坑记录

实际落地中会遇到几个容易忽略的点：

1. **符号链接绕过**  
   如果仅用 `Path.absolute()` 或 `os.path.abspath()`，符号链接指向白名单外的文件可能被通过。必须使用 `resolve()` 或 `os.path.realpath()` 解析最终目标。

2. **相对路径与当前工作目录（CWD）**  
   检查时的路径解析依赖 CWD。如果 CWD 被修改，同一个相对路径可能指向不同位置。建议始终将传入路径转为绝对路径后再检查；或在护栏中强制所有相对路径相对于固定工作区 root。

3. **临时文件与系统目录**  
   `tempfile.mkdtemp()` 和 `tempfile.NamedTemporaryFile()` 默认在系统临时目录创建，不在白名单内就会失败。需要手动指定 `dir` 参数到白名单目录，或者在白名单里加入相应的临时目录。

4. **`shutil.rmtree` 与多重操作**  
   删除目录树可能包含符号链接指向外部，需要自行遍历并逐个检查，或者直接禁止 `rmtree`，改用安全的工具函数递归删除（只删白名单内的文件/目录）。

5. **性能开销**  
   每次 I/O 都做路径解析和字符串比对，低频场景毫无影响。高频（如大量小文件读取）建议做路径缓存，将绝对路径 -> bool 的结果存入字典（注意 CWD 变化缓存失效）。

---

## 可复用建议

- **做成独立的安全层模块**，用 `pip` 安装到 Agent 环境。提供 `enable_guard()` 函数一键注入安全函数。
- **配合环境变量动态配置**白名单路径，避免硬编码。
- **记录所有拒绝日志**，集中监控，可及时发现异常调用模式，比如绕过尝试。
- **结合测试用例**：写一组越界路径（`../etc`，符号链接出圈，绝对路径 `/etc/passwd`）确保被拦截；正常白名单内操作不受影响。
- **对 Agent 的输出也做检查**：若 Agent 输出命令供外部执行，需使用白名单限制命令行操作，但那是另一层（进程沙箱），本文先聚焦文件系统。

---

## 总结

即使是内部自动化，给 Agent 添加**最低权限原则**是值得的。  
为文件访问加一个目录白名单，实现成本仅几十行代码，却能在误操作、恶意输入甚至自身 Bug 面前提供一层关键保护。  
别等文件被改写、数据丢失后才补护栏——从第一个自动化脚本开始，就把安全当作基础设施。

在 OpenClaw 生态里，工具函数和插件往往是信任计算链条的一环，我们自己就是那个“给权限”的人。少给一点权限，多一份安稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/5008ad5c36190b64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/7edd3b17a7e3efc6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/05f1c8aba822c2e4.png)

