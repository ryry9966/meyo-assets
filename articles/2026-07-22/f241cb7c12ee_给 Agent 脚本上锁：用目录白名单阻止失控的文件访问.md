---
title: 给 Agent 脚本上锁：用目录白名单阻止失控的文件访问
feedId: 30093
source: 综合讨论
publishedAt: 2026-07-22
---

# 给 Agent 脚本上锁：用目录白名单阻止失控的文件访问

## 一、背景

在 OpenClaw 这类 Agent 框架里，MCP 工具和自定义脚本可以让 LLM 直接操作本地文件系统。这种能力在“文档整理”“日志分析”“批量重命名”等场景下确实高效，但也带来了最让人不安的问题：一旦 prompt 被注入、模型幻觉或者逻辑分支意外跳转，Agent 可能读取、篡改甚至删除你不想暴露的文件。

常见的保护手段是“全部禁止，按需挂载”，比如 Docker volume 映射，或者把工作目录限定在一个沙箱里。但本地脚本运行时经常需要访问多个散落目录：配置文件在 `~/.config/app`，数据在 `~/Documents/project`，临时输出到 `/tmp`。完全沙箱化会牺牲灵活性，每次切换项目都要重新挂载，很不方便。

一个工程上更平衡的做法是：**在 Agent 进程层实施目录白名单**，允许脚本只读写明确指定的路径，其余一律拒绝。本文介绍如何用 Python 的装饰器和路径解析实现一个轻量的文件访问护栏，可直接接入 OpenClaw 的工具函数。

## 二、问题定义

假设你已经用 OpenClaw 的 MCP 暴露了一个本地 shell 工具，Agent 可以调用它执行任意命令。即使你限制了命令列表，只要允许 `cat`、`echo`、`rm` 等基础命令，路径参数仍是不可控的。

简单的安全模型应该是：

- 声明一个全局白名单，包含允许读写的目录列表。
- 所有涉及文件 I/O 的工具函数（无论是 shell 命令封装还是 Python 原生 open）都必须先通过路径检验。
- 非法访问立即拒绝，并记录告警日志。

实现时需要注意符号链接绕过、相对路径绕过、以及 `..` 路径穿越。

## 三、实现步骤

### 1. 目录白名单与路径规范化

定义一个 `PathGuard` 类，负责判断给定路径是否在允许的目录树内。核心是使用 `os.path.realpath` 解析符号链接和相对路径，再检查前缀匹配。

```python
import os
from pathlib import Path
from typing import List

class PathGuard:
    def __init__(self, allowed_dirs: List[str]):
        self.allowed_dirs = [os.path.realpath(d) for d in allowed_dirs]

    def is_allowed(self, target: str) -> bool:
        real_target = os.path.realpath(target)
        for base in self.allowed_dirs:
            if real_target.startswith(base + os.sep) or real_target == base:
                return True
        return False
```

注意：`startswith` 需要精确到目录分隔符，防止 `/var/app` 匹配 `/var/app2`。

### 2. 装饰器注入护栏

对 Agent 工具函数使用装饰器，在执行前检查所有字符串参数中可能出现的路径。一种简单策略是要求函数显式标注哪些参数是文件路径，例如用类型注解 `PathArg`。

```python
from functools import wraps
from typing import get_type_hints
import shlex

class PathArg(str):
    pass

def guarded(guard: PathGuard):
    def decorator(func):
        hints = get_type_hints(func)
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 根据提示找到 PathArg 类型的实参
            arg_names = func.__code__.co_varnames[:func.__code__.co_argcount]
            for name, value in zip(arg_names, args):
                if hints.get(name) == PathArg:
                    if not guard.is_allowed(value):
                        raise PermissionError(f"Access denied: {value}")
            for name, value in kwargs.items():
                if hints.get(name) == PathArg and not guard.is_allowed(value):
                    raise PermissionError(f"Access denied: {value}")
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

Shell 命令封装中，路径往往混在命令字符串中。更稳妥的办法是**只允许白名单路径作为工作目录**，并在执行前切换到该目录，同时禁止 `cd` 命令。但更好的工程实践是尽量避免拼接命令字符串，而是直接用 `subprocess` + 参数列表 + `cwd` 参数，然后用 guard 检查 `cwd`。

例子：一个只允许在 `/tmp/sandbox` 和 `/data/projectA` 读写的 cmd 工具：

```python
guard = PathGuard(["/tmp/sandbox", "/data/projectA"])

@guarded(guard)
def run_cmd(command: str, cwd: PathArg = "/tmp/sandbox"):
    # cwd 已经被检查
    result = subprocess.run(command, shell=True, cwd=cwd, capture_output=True, text=True)
    return result.stdout
```

`cwd` 是 `PathArg` 类型，装饰器会自动校验。

### 3. 集成到 OpenClaw 的 MCP 工具中

OpenClaw 的 MCP 工具本质上是带有 `@tool` 装饰器的异步或同步函数。直接把 `guarded` 包装在工具函数外层即可。例如：

```python
from openclaw.tools import tool

@tool
@guarded(guard)
async def read_file(path: PathArg) -> str:
    with open(path, 'r') as f:
        return f.read()
```

注意装饰器顺序：`@tool` 应该在最外层，以保证 MCP 发现的是被 guard 包裹的函数。

### 4. 日志与监控

在 `PathGuard.is_allowed` 返回 `False` 时，不仅抛出异常，还要记录包含时间、调用栈、请求路径的结构化日志，方便事后分析 Agent 行为。建议使用 `logging` 输出到独立文件，便于监控采集。

## 四、踩坑记录

**符号链接穿透**  
用户可能在白名单目录下创建一个指向 `/etc/passwd` 的软链接。`realpath` 解析后路径会落在 `/etc`，不在白名单内，因此会被拒绝。但如果文件是硬链接，`realpath` 检查依然有效，因为硬链接解析后仍是同一个 inode，路径本身已经是真实路径。注意：如果你允许 Agent 创建符号链接，需要额外检测链接目标，可以引入 `os.lstat` 判断。

**`/tmp` 目录下的竞争条件**  
`/tmp` 目录通常是全局可写，Agent 可以读到其他用户的临时文件。务必使用 `tempfile.mkdtemp` 创建 Agent 专用子目录，并将 `/tmp` 白名单限定为该专用目录，而不是整个 `/tmp`。

**路径参数注入**  
即使路径被检查，命令本身可能包含反引号或 `$()` 来执行子命令。避免 `shell=True` 是最好的选择。如果必须用 shell，需要结合 `shlex.quote` 对参数转义，但这并不绝对安全。强烈建议换成 `subprocess.run([...])` 传参方式。

**相对路径与工作目录不一致**  
Agent 可能通过修改 `cwd` 让 `../` 指向白名单外。我们的 guard 检查的是最终 `realpath`，不管怎么绕，真实路径都会暴露。但最好也强制限制 `cwd` 必须在白名单内，并拒绝任何含 `..` 的相对路径参数。

## 五、可复用建议

- **白名单最小化**：只开放真正需要的目录，且每个 Agent 实例使用独立白名单。例如按项目维度创建多个 guard 实例。
- **只读与读写分离**：扩展 `PathGuard` 增加操作类型（读/写/执行），将写操作限制在更小的临时输出目录里。
- **封装为标准中间件**：如果你使用 OpenClaw 的 middleware 机制，可以编写一个 `PathAccessMiddleware`，在工具调用前统一检查，无需逐个工具加装饰器。
- **与文件系统 ACL 结合**：在 Linux 上用 `setfacl` 额外限定 Agent 进程用户的权限，双重保险。
- **测试用例**：提前构造符号链接、相对路径穿越、硬链接等攻击场景，确保 guard 在 CI 中被持续验证。

## 六、总结

给 Agent 脚本加目录白名单，本质上是用工程化的约束来弥补 LLM 不可预测的行为。这个护栏不会影响正常功能，但能有效阻止灾难性的文件越权操作。几十行代码就可以实现，并可以无缝集成到 OpenClaw 的工具层。

在信任模型上，我们无法假设 prompt 永远安全，也无法假设模型不会“发疯”。所以，只能从最底层的 I/O 通道入手，替 Agent 画一个明确的牢笼。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/5be8a618a18d32d4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/342d0b53687fe25c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/406c6acaae5eab2d.png)

