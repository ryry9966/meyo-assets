---
title: 给自动化脚本加一把“目录锁”：Agent 文件访问护栏的工程实践
feedId: 29772
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景：当 Agent 开始“乱翻”你的文件系统

在 OpenClaw 这类 Agent 平台中，我们经常会编写或接入自动化脚本工具，让 Agent 能够操作本地文件——比如把下载的文件归类到指定目录、读取日志、生成报告。这些脚本通常会被包装成插件或 MCP 工具，在 Agent 的“大脑”触发下运行。

问题在于，Agent 是根据模型输出来决定调用哪个工具的，而模型输出天然存在不确定性。如果脚本本身没有做任何权限约束，一次非预期的调用就可能读到敏感文件（如 `~/.ssh` 下的密钥），甚至覆写系统配置。即便你的 Agent 目前只在“受控环境”运行，随着自动化程度的加深，这种开放式的文件访问迟早会成为隐患。

因此，我们需要给这些自动化脚本配上一把**目录锁**：只允许读写在预先声明的白名单路径内，其余位置一律拒绝。这比单纯的“不要写危险路径”更可靠，因为它从根本上缩小了操作面。

## 问题拆解：什么样的护栏才算“有效”

一个只做了字符串前缀匹配的检查很容易被绕过，如下面的路径：

```
/project/data/../../etc/passwd
```

如果脚本只是判断路径是否以 `/project/data` 开头，这个相对路径穿越就能轻松逃逸。所以我们要的不是简单的字符串前缀，而是**规范化后的绝对路径对比**。

在此基础上，一个可用的文件访问护栏需要满足：

1. 对传入的所有路径做规范化：解析符号链接、剔除 `..` 和 `.`。
2. 检查规范化后的路径是否在白名单目录内（即路径是白名单目录的子路径）。
3. 同时覆盖读和写操作：`open(path, 'r')` 和 `open(path, 'w')` 都要被限制。
4. 对脚本开发者友好，尽量不改动大量历史代码。

在 Python 环境下，最趁手的方案是写一个**带路径检查的 Context Manager 或包装函数**，用 `os.path.realpath` 解析后再用 `os.path.commonpath` 做前缀判断。下面是一个可复用的实现思路。

## 做法：用 `os.path.realpath` 和 `commonpath` 构建白名单门禁

### 1. 定义白名单与检查函数

先声明允许访问的目录列表，比如：

```python
ALLOWED_DIRS = [
    "/home/agent/project/data",
    "/tmp/agent_workspace"
]
```

然后写一个校验函数：

```python
import os

def is_path_allowed(path: str, allowed_dirs: list[str]) -> bool:
    # 先拿到绝对路径，但此时可能包含 .. 或符号链接未解析
    given_abs = os.path.abspath(path)
    # 解析符号链接并去掉 .. 和 .
    real_path = os.path.realpath(given_abs)
    # 如果 real_path 就是某个 allowed_dir 的子路径，则通过
    for allowed in allowed_dirs:
        # 规范化 allowed 目录本身（解析可能的符号链接）
        real_allowed = os.path.realpath(allowed)
        # commonpath 比较公共前缀
        if os.path.commonpath([real_path, real_allowed]) == real_allowed:
            return True
    return False
```

注意：`os.path.realpath` 在本路径尚未存在时仍会解析目录部分的符号链接，但最终不存在的文件部分会被保留。这通常够用。如果你需要检查“待创建文件”的父目录是否合法，可以取 `os.path.dirname(real_path)` 再做目录存在性判断，但白名单机制本身已经可以拦截。

### 2. 封装安全版 `open`

我们可以写一个 `safe_open` 函数，行为与内置 `open` 相似，但在打开前先做路径检查。示例：

```python
class FileNotAllowedError(PermissionError):
    pass

def safe_open(path: str, mode: str = 'r', *args, **kwargs):
    if not is_path_allowed(path, ALLOWED_DIRS):
        raise FileNotAllowedError(f"Access to {path} is not allowed.")
    return open(path, mode, *args, **kwargs)
```

如果需要支持 `os.listdir`、`os.mkdir`、`shutil.move` 等其他文件操作，可以类似地包装。但考虑到工作量，更务实的做法是只在关键的入口处强制使用 `safe_open`，而对目录遍历、临时文件等操作，在明确理解风险的前提下用白名单目录 + 规范化路径来调用。

### 3. 集成到插件或工具脚本

假设你有一个 OpenClaw 插件，其中有一个 `run_daily_report` 函数，原本直接用 `open` 写文件：

```python
def run_daily_report(output_path: str, content: str):
    with open(output_path, 'w') as f:
        f.write(content)
```

改造后变为：

```python
def run_daily_report(output_path: str, content: str):
    with safe_open(output_path, 'w') as f:
        f.write(content)
```

只要在调用该函数前，Agent 传入的 `output_path` 被 `safe_open` 检查，就能确保最终写入位置不会超出白名单。

## 踩坑记录：符号链接、竞争条件与 macOS 差异

在实际落地过程中，几个坑点值得注意：

1. **符号链接指向白名单外路径**  
   如果用户在允许目录内创建了一个指向 `/etc` 的符号链接 `data/link_to_etc`，`os.path.realpath` 会把它解析到 `/etc`，我们的检查会拒绝访问。这正是我们希望的行为。但你需要知道这一点，因为有的开发者可能预期“链接在我的工作目录下就应该放行”。这时候需要明确规范：白名单解析后比对，链接源也必须在允许范围内。

2. **文件不存在时 `realpath` 的行为**  
   在 Linux 下，如果路径对应的文件不存在，`os.path.realpath` 会解析所有存在的目录部分，对不存在的文件名保留。因此，你想在允许目录下创建一个尚不存在的文件，检查其父目录是否合法即可。通常我们直接用 `realpath` 对比，因为子目录一定在白名单下，即使文件不存在，只要目录结构合法就行。不过有一个边界：如果父目录是符号链接，且链接指向外部，那么 `realpath` 就会暴露真实路径，从而被拦截——这也是合理的。

3. **竞争条件（TOCTOU）**  
   在检查通过与实际操作之间，路径可能被替换。但对我们这种本地自动化场景，这种攻击面的概率极低，通常不必过度设计。如果你很不放心，可以在打开文件后使用 `f.fileno()` 和 `os.stat` 再次确认，但工程代价较大。

4. **macOS 上 `/tmp` 是符号链接**  
   `/tmp` 在 macOS 下是指向 `/private/tmp` 的符号链接。如果你把允许目录写成 `["/tmp/agent_workspace"]`，那么解析后会出现 `/private/tmp/agent_workspace`，需要确保白名单目录的写法与实际解析一致。建议白名单中直接使用真实路径，或者在初始化时用 `os.path.realpath` 归一化一次。

## 可复用建议：抽象为一个可插拔的 Policy

考虑到未来可能需要对网络访问、进程执行也加白名单，建议将文件访问控制抽象为一个独立的 Policy 模块：

```python
class FileAccessPolicy:
    def __init__(self, allowed_dirs: list[str]):
        self._allowed_real = [os.path.realpath(d) for d in allowed_dirs]

    def is_allowed(self, path: str) -> bool:
        given = os.path.abspath(path)
        real = os.path.realpath(given)
        for allowed in self._allowed_real:
            if os.path.commonpath([real, allowed]) == allowed:
                return True
        return False

    def require(self, path: str):
        if not self.is_allowed(path):
            raise PermissionError(f"Access denied: {path}")
```

然后在你的脚本上下文初始化一个全局的 `fs_policy`，所有工具函数都通过它来校验。这样即使以后需要对接多个插件，也只维护一份白名单和策略。

## 总结：一个低成本但有效的安全增量

给自动化脚本加目录白名单，本质上是在信任边界上添加一层最小权限约束。它无法防御内核漏洞或恶意提权，但能让 Agent 脚本从“漫天飞”变成“只走规定路线”。对大多数本地自动化场景，这种检查带来的性能开销微乎其微，却能显著降低误调用、模型幻觉引发的脚本“越界”风险。

如果你的 OpenClaw 插件会拿着 Agent 给出的路径去操作文件，不妨今晚就把这段 `safe_open` 加入代码库。下一次当模型试图聪明地“帮你看看 `/etc/shadow`”时，你会在日志里看到一条干净的错误，而不是一身冷汗。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/df4fb93305997ead.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/e33ad468b3cd6080.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/5a738f9615268d72.png)

