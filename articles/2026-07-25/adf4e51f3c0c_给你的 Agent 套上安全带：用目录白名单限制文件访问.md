---
title: 给你的 Agent 套上安全带：用目录白名单限制文件访问
feedId: 30352
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景：当自动化脚本拥有了文件系统权限

在 Agent 实践里，越来越多的场景需要脚本直接读写文件——导出报告、缓存上下文、保存用户上传的资源。无论是 MCP 服务器实现、OpenClaw 自动化动作，还是你自己写的工具插件，一旦脚本获得文件系统访问权，很容易变成一个“到处乱翻的幽灵”。即使初衷只是读写 `/tmp/my_cache`，试想以下代码：

```python
with open("../../../etc/passwd", "r") as f:
    pass
```

在没有任何保护的情况下，一个恶意构造的相对路径就能穿透到敏感目录。如果你把这种脚本部署在生产环境或者对外开放的 Agent 工作流里，风险不言而喻。

工程上，我们当然可以用系统级的沙箱（如 Docker、seccomp、AppArmor）来兜底，但在 Agent 工具层直接实现一个轻量的目录白名单护栏，成本更低、反馈更快，并且可以作为“纵深防御”的一环。

## 问题定义：允许访问哪些路径，其余一律拒绝

我们要实现的效果很简单：定义一个白名单目录列表（例如 `["./workspace", "/tmp/myagent"]`），然后确保所有文件 I/O 操作的目标路径都必须落在这几个目录内（或其子目录）。一旦检测到越界，直接抛出异常。

看似简单，但做起来有几个隐蔽的坑：

- 相对路径问题：`./workspace/../secrets/key` 实际指向 `./secrets/key`，可能跳出白名单。
- 符号链接绕过：`workspace/link -> /etc`，打开 `workspace/link/passwd` 实际上访问了 `/etc/passwd`。
- 路径规范化差异：不同操作系统对尾部斜杠、大小写（Windows）的处理不一致。
- 路径拼接漏洞：代码里用 `allow_dir + user_input` 拼接，如果 `user_input` 以斜杠开头会改变路径根。

## 实现步骤：一个可复用的路径检查器

### 1. 核心检查函数

Python 标准库已经提供了足够的武器。核心思路：把待检查的路径解析为**真实绝对路径**，然后判断它是否以任一白名单目录（同样解析为真实绝对路径）为前缀。

```python
import os
from pathlib import Path
from typing import List

class PathGuard:
    def __init__(self, allowed_dirs: List[str]):
        # 预先将白名单目录解析为真实绝对路径
        self._allowed_real = [
            os.path.realpath(Path(d).resolve(strict=False))
            for d in allowed_dirs
        ]

    def check(self, given_path: str) -> str:
        # 1. 转为绝对路径（处理相对路径）
        abs_path = os.path.abspath(given_path)
        # 2. 解析符号链接与“..”得到规范的真实路径
        real_path = os.path.realpath(abs_path)

        # 3. 检查 real_path 是否在任意白名单目录下
        allowed = any(
            real_path.startswith(prefix + os.sep) or real_path == prefix
            for prefix in self._allowed_real
        )
        if not allowed:
            raise PermissionError(f"Access denied: {given_path}")
        return real_path   # 返回规范路径，后续操作都使用它
```

注意几个细节：
- `os.path.abspath` 处理相对路径，把 `../../x` 变成基于当前工作目录的绝对路径。如果当前工作目录不可控，你可以在初始化时锁定工作目录，或者强制要求在检查前设置好当前目录。
- `os.path.realpath` 解析链路中的所有符号链接，是防绕过的关键。
- 前缀匹配时要注意 `startswith` 的陷阱：`/home/user` 不等于 `/home/user01`，所以我们加上了 `os.sep` 且单独判断完全相等的情况。

### 2. 内置文件操作包装

单有检查器还不够，你需要让实际的文件操作调用 `check`。推荐两种封装方式：

**装饰器拦截**

对标准库函数做包装，返回一个安全的替代版本：

```python
import builtins

def safe_open(guard, original_open):
    def wrapper(file, mode='r', *args, **kwargs):
        file = guard.check(file)
        return original_open(file, mode, *args, **kwargs)
    return wrapper

# 注入到脚本的全局命名空间
builtins.open = safe_open(guard, builtins.open)
```

这种方式侵入性强，但能透明化保护现有脚本。

**工具函数**

或者提供一个显式的 `safe_read`、`safe_write`，让同事在编写工具时主动调用：

```python
def safe_write(guard, path, content):
    real_path = guard.check(path)
    with open(real_path, 'w') as f:
        f.write(content)
```

后者更适合团队协作，因为白名单行为是显式的，不容易被无意绕过。

### 3. 集成到 MCP 工具与 Agent 动作

如果你在写 MCP server（比如基于 `mcp` Python 包），可以在每个工具处理函数入口处调用 `guard.check`。举个例子，文件读取工具：

```python
from mcp.server import Server, Tool

guard = PathGuard(allowed_dirs=["./workspace", "/tmp/agent_io"])

async def read_file(path: str) -> str:
    real_path = guard.check(path)
    with open(real_path) as f:
        return f.read()
```

对于 OpenClaw 的自定义动作脚本，同样可以在脚本模版里预置一个 `check_path` 函数，并文档化要求所有文件调用必须经过它。

## 踩坑记录

1. **符号链接仍可能漏网**  
   如果在检查后到实际打开文件之间，文件被替换为符号链接（TOCTOU 竞态条件），仍然会被绕过。轻量护栏无法解决竞态问题，需要系统级手段。但在大多数 Agent 场景下，这种攻击窗口极小且攻击者难以利用，可以接受。

2. **多进程/多线程的工作目录不同**  
   `os.path.abspath` 依赖于当前工作目录。如果脚本或不同线程修改了当前目录，检查结果可能不符合预期。一种做法是禁止依赖相对路径，强制要求传入绝对路径，并在 `check` 里检测是否为绝对路径，否则直接报错。这样可以彻底消除相对路径的副作用。

3. **Windows 上的路径分隔符与驱动器号**  
   `startswith` 在比较时需要统一大小写。可以用 `os.path.normcase` 处理白名单和待检查路径。同时注意 `os.path.realpath` 在 Windows 上也会得到包含盘符的长路径，需要注意 UNC 路径等。不过 OpenClaw 社区大多以 Linux 环境为主，可先忽略。

4. **性能开销**  
   每次文件操作都调用 `realpath`，在高频 I/O 场景下可能有性能影响。你可以为白名单目录预先缓存已解析路径，并在 `realpath` 调用时利用文件系统缓存。通常 Agent 工具脚本的 I/O 并不密集，这点开销可以忽略。

## 可复用建议

- **封装成独立库**：把 `PathGuard` 和对应的 `safe_open` 等函数打包到你的内部工具库中，其他同事只需要 `pip install` 即可复用。
- **配置化白名单**：通过环境变量或配置文件传入白名单目录，避免硬编码。例如：`ALLOWED_DIRS=/app/workspace,/tmp/agent_cache`。
- **与现有权限模型结合**：如果你的 Agent 框架有角色或上下文概念，可以把白名单和角色绑定，比如只有“报告导出”动作可以访问 `/exports`。
- **日志与监控**：每次拒绝访问时记录日志，方便审计和排错。在生产中，这能帮你发现误配置或被攻击的迹象。

## 总结

一个只有几十行代码的目录白名单检查器，就能让你的 Agent 脚本从“裸奔”变成“有安全带的驾驶”。它不能替代完整沙箱，但作为纵深防御的第一道关卡非常有效，尤其适合内部工具、自动化流水线、研发实验环境。

工程实践的关键在于：尽早解析真实路径、避免相对路径依赖、把检查逻辑集中在可控的少数函数里。做到这三点，就能显著降低路径穿越和意外读写敏感文件的风险。

下次当你写一个需要操作文件的 MCP 工具时，不妨先把 `PathGuard` 嵌进去——它可能是你今天最值得花的 30 分钟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/9310859ef3273b9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/35c05f66be28c677.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/370d49eb05d6d2ff.png)

