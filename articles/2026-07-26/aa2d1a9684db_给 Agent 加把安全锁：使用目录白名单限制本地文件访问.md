---
title: 给 Agent 加把安全锁：使用目录白名单限制本地文件访问
feedId: 30530
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：Agent 的“自由”过了火

在 OpenClaw 这类 Agent 框架里，我们经常会给模型挂载本地工具，比如读写文件、遍历目录、执行 Shell 命令。这在自动化脚本场景里特别常见——比如用 Agent 批量整理下载文件夹，把 PDF 分类归档，或者根据 Excel 生成周报。

问题在于，这些工具一旦挂上去，模型就能以当前进程的权限接触整个文件系统。你原本只想让它操作 `~/Downloads`，但它完全可能在一次错误的推理链里，读到 `~/.ssh/id_rsa`，甚至覆盖掉 `.bashrc`。而且这种错误往往不是恶意注入，更多是 prompt 不够精确、模型遗忘约束、或者上下文过长导致的行为漂移。

所以，我们需要一种工程化手段，在工具层就卡死边界：**允许访问哪些目录，就只能“看见”这些目录**。这就是所谓的“目录白名单”护栏。

## 做法：三步实现文件访问拦截

下面以 OpenClaw 常见的工具定义方式为例（Python 环境），展示一个可复用的实现路径。核心思路是：对所有文件相关工具的参数路径做一次规范化检查，只有落在白名单内的操作才放行。

### 步骤 1：定义白名单并实现路径检查函数

在项目配置里，明确定义允许读写的目录列表（绝对路径）：

```python
ALLOWED_PATHS = [
    "/home/op/Downloads",
    "/home/op/projects/safe_area",
    "/tmp/agent_workspace"
]
```

然后写一个检查函数，重点在于**先规范化路径再匹配**，而不是简单的字符串前缀判断。否则符号链接、`..`、相对路径等都会成为逃逸通道。

```python
import os

def is_path_allowed(target: str) -> bool:
    # 规范化到真实绝对路径，消除链接和相对引用
    real = os.path.realpath(os.path.abspath(target))
    for allowed in ALLOWED_PATHS:
        allowed_real = os.path.realpath(allowed)
        # 必须是 allowed 目录下的子孙路径，或本身相等
        if real == allowed_real or real.startswith(allowed_real + os.sep):
            return True
    return False
```

这里的几个要点：

- 用 `os.path.realpath` 解决符号链接（比如 `~/Downloads -> /mnt/data`，如果 `/mnt/data` 不在白名单就会被拒）。
- 用 `startswith + os.sep` 防止 `/tmp/my` 被 `/tmp/my_evil` 误匹配（目录名重叠）。
- 两边都做规范化，避免白名单里写的是相对路径或含链接的情况。

### 步骤 2：以装饰器形式集成到工具定义中

在 OpenClaw 里，工具通常是加了 `@tool` 装饰器的函数。我们可以再包一层装饰器，在执行前做白名单检查：

```python
from functools import wraps

def require_whitelisted_path(arg_name="path"):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            target = kwargs.get(arg_name)
            if target is None:
                raise ValueError(f"Missing path argument '{arg_name}'")
            if not is_path_allowed(str(target)):
                raise PermissionError(f"Access denied: {target}")
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

然后把它套在具体的工具上：

```python
@tool
@require_whitelisted_path("file_path")
def read_file(file_path: str) -> str:
    with open(file_path, "r") as f:
        return f.read()
```

对于 `write_file`、`list_directory` 等工具，也一样处理。OpenClaw 的 MCP 工具或外部插件也可以用类似思路，在工具描述生成前，或者在调用包装器中植入。

### 步骤 3：在进入 Agent 前统一校验

如果项目里工具数量多，不想逐个装饰，可以在工具注册环节统一处理。比如 OpenClaw 的自定义 ToolWrapper 机制：

```python
class WhitelistedToolWrapper:
    def __init__(self, tool, path_arg="path"):
        self.tool = tool
        self.path_arg = path_arg

    def __call__(self, **kwargs):
        if self.path_arg in kwargs:
            if not is_path_allowed(kwargs[self.path_arg]):
                raise PermissionError(f"Access denied: {kwargs[self.path_arg]}")
        return self.tool(**kwargs)
```

然后在工具列表构建时，自动包装所有含路径参数的工具。这样对已经定义好的工具函数侵入性更小。

## 实战踩坑点

1. **符号链接逃逸**  
   只做 `abspath` 不够，一定要 `realpath`，否则软链接会直接绕过前缀检查。我遇到过用户把 `~/agent_data` 链到 `/home/user/.hidden` 的情况，结果模型读了隐藏目录。

2. **Windows 路径分隔符**  
   如果在 Windows 上跑 Agent，`os.sep` 会自动为 `\`，但白名单配置可能混用 `/`。可以用 `os.path.normcase` + `os.path.normpath` 统一处理，但这会牺牲一点跨平台简洁性。更好的做法是统一配置成 `pathlib.Path` 对象，用 `Path.resolve()` 代替 `os.path.realpath`。

3. **临时文件与日志**  
   如果 Agent 内部需要用临时文件，而白名单不含 `/tmp`，那就会失败。需要在白名单里显式加入 `tempfile.gettempdir()`，或者在工具实现里为临时操作单独豁免（不推荐豁免，容易松动原则）。

4. **性能**  
   `realpath` 会做文件系统调用，但文件操作本身就是 I/O 密集的，这点开销在 Agent 场景里可以忽略。尽量在检查前做缓存（比如对同一个路径短时间内只检查一次），可以在高并发时减少重复 stat 调用。

## 可复用建议

- **配置文件化**：把 `ALLOWED_PATHS` 写在 YAML/TOML 里，与环境或用户绑定，支持通配符（如 `~/agents/*`），在加载时展开。
- **与权限通知结合**：当工具因白名单拒绝时，把拒绝信息写成明确的自然语言错误，返回给模型，让它知道“你无权访问”，避免无限重试。
- **多级沙箱**：对于更严格的需求，考虑将读写分离——读白名单可以大一些，写白名单只限定在临时工作区。这个可以通过两个装饰器实现：`@read_whitelist` 和 `@write_whitelist`。
- **勿忘目录列表**：仅仅限制读写还不够，`list_directory` 也要加白名单，否则模型能看见整个根目录结构，信息量同样很大。

## 总结

给 Agent 加文件访问护栏，本质上不是防攻击，而是防“意外”。把白名单这种老派权限模型用在 LLM 工具层，成本低、见效快，而且符合最小权限原则。核心代码不到 30 行，却能把一次潜在的隐私泄漏变成 `PermissionError`。

在 OpenClaw 这类鼓励工具生态的框架里，越早把这些边界写进工具定义模板，后续大规模自治运行的收益越大——因为你不再是“祈祷模型别乱碰文件”，而是用代码锁死了它能碰的范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/f68b17e75faa655b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/9ed92e2856219c72.png)

