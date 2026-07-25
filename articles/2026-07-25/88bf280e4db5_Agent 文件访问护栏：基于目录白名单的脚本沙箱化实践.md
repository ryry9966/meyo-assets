---
title: Agent 文件访问护栏：基于目录白名单的脚本沙箱化实践
feedId: 30372
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 OpenClaw 这类可扩展 Agent 平台里，让 Agent 调用本地命令或读写文件的场景越来越多。无论是直接执行 Shell 脚本，还是通过 MCP 工具暴露文件操作，Agent 在无人监督时很容易触碰到不该碰的路径。一个自主决策删除旧日志的脚本，可能不小心把系统配置文件、密钥或整个用户目录删掉。与其在事后庆幸有备份，不如在 Agent 执行环境里加一道“文件访问护栏”，让每次读写只发生在明确允许的目录内。

这篇文章记录一个工程化程度较高、可直接嵌入自定义 Tool 或 MCP Server 的目录白名单实现，重点放在真实踩坑点上，不依赖额外的沙箱框架，仅靠 Python 标准库就能跑起来。

## 问题拆解

一个典型的 Agent 工具函数（比如`run_shell`）拿到指令后，可能直接调用`open()`或者`os.remove()`，路径完全由 Agent 的输出决定。潜在风险包括：

- **路径穿越**：`../../etc/cron.d` 跳转到意料之外的系统目录。
- **符号链接绕过**：白名单目录内若存在指向外部的软链接，`open()`会顺藤摸瓜访问到受限文件。
- **相对路径与工作目录耦合**：Agent 服务重启后 CWD 变化，相对路径可能指向完全不同的位置。
- **并发 TOCTOU**：检查通过后到实际操作前，路径被替换（虽然低概率，但在高权限进程里仍需留意）。

这些风险归结为一个核心需求：**在执行任何文件 I/O 前，必须把传入路径解析为规范化的绝对真实路径，然后判断它是否以白名单中的目录前缀开头。**

## 实现步骤

### 1. 定义白名单配置

把白名单目录作为列表，通过环境变量传递，并统一转换为绝对路径。这样做的好处是不用改代码就可以在不同部署环境里调整。

```python
import os

SAFE_ROOTS = [
    "/app/output",
    "/app/logs",
    "/tmp/agent_workspace",
    # 从环境变量读取额外目录
]
# 确保所有白名单目录本身也是绝对、真实路径
SAFE_ROOTS = [os.path.realpath(d) for d in SAFE_ROOTS]
```

### 2. 编写路径检查核心函数

核心逻辑：先用`os.path.realpath()`解析所有符号链接并规范化，然后判断是否以任一白名单路径开头。必须同时处理目录与普通文件，并且考虑到路径分隔符，结尾加上分隔符能防止“`/tmp/app`被误当成`/tmp/app-data`的前缀”这种情况。

```python
def is_path_allowed(user_path: str) -> bool:
    try:
        real_path = os.path.realpath(user_path)
    except OSError:
        return False    # 无法解析的路径直接拒绝

    for root in SAFE_ROOTS:
        # 加上分隔符确保精确前缀匹配
        if real_path.startswith(root + os.sep) or real_path == root:
            return True
    return False
```

注意：这里没有简单地用`startswith(root)`，否则`/app/output_bak`会被误认为是`/app/output`的子目录。加分隔符是最小成本的修正。

### 3. 对常用 I/O 操作做统一包装

不建议在每个工具函数里重复编写检查逻辑。可以提供一个`SafeFileOps`类，将`open`、`rmtree`等操作封装一次，后续所有 Agent 工具都通过它来访问文件。

```python
import builtins
import shutil

class SafeFileOps:
    @staticmethod
    def open(file, mode='r', *args, **kwargs):
        if not is_path_allowed(file):
            raise PermissionError(f"Access to {file} is denied by whitelist")
        return builtins.open(file, mode, *args, **kwargs)

    @staticmethod
    def remove(file):
        file = os.path.realpath(file)
        if not is_path_allowed(file):
            raise PermissionError(f"Remove {file} denied")
        os.remove(file)

    # 按需补充 stat, listdir, mkdir 等...
```

这样一来，Agent 工具函数里只需要 `SafeFileOps.open(path, 'w')` 就可以安全写入了。

### 4. 集成到 MCP Server 或自定义 Tool

在 OpenClaw 的工具配置中，将原有的`open`或`os.write`调用全部替换为`SafeFileOps`。如果是通过`subprocess`执行命令行工具，可以考虑将工作目录切换到白名单目录下，并额外对环境变量`PATH`做限制，但更好的做法是把命令行输入也做路径审计，或者干脆让脚本只通过封装后的 API 操作文件。

## 踩坑记录

- **符号链接双向欺骗**：`realpath`会解析最终目标，但如果白名单目录自身就是一个符号链接，需要确保配置时也使用`realpath`，否则可能出现白名单路径为`/data`（软链到`/mnt/disk1`），而解析后路径为`/mnt/disk1`，前缀匹配失败。所以初始化`SAFE_ROOTS`时务必也做一次`realpath`。
- **Windows 盘符与路径分隔符**：在 Windows 下`os.sep`为反斜杠，`realpath`返回的路径也带盘符，处理逻辑同样适用，但需要注意大小写敏感性。`realpath`在 Windows 上不会自动统一大小写，需要结合`os.path.normcase`实现大小写不敏感匹配。
- **相对路径陷阱**：用户传入`file.txt`，`os.path.realpath`会相对于当前工作目录解析，因此得到的绝对路径与工作目录关联。如果工作目录是白名单内的目录，那么结果是允许的；但如果工作目录是根目录，可能就变成`/root/file.txt`被禁掉。这通常是预期行为，但要明确告知使用者，或者统一要求工具函数内部将相对路径转换为相对于某个白名单根目录的路径。
- **`open()`内部的符号链接跟随**：`builtins.open`默认跟随符号链接，`SafeFileOps.open`在检查通过后才执行`open`，理论上检查时解析了一次，`open`时又解析一次，没有 TOCTOU 防范能力。高安全场景下可换用`open(path, O_RDONLY | O_NOFOLLOW)`并配合`fdopen`，但这种做法会改变代码习惯，目前多数场景用白名单加低权限用户就能覆盖。

## 可复用建议

- **装饰器化**：为工具函数写一个`@requires_allowed_path(arg_index)`装饰器，自动校验指定参数，减少重复代码。
- **配置驱动**：白名单从`config.yaml`或环境变量加载，方便在不同 Agent 实例间复用。
- **审计日志**：在`is_path_allowed`返回`False`时记录危险调用详情，帮助发现 Agent 逻辑偏差或攻击尝试。
- **与系统层结合**：如果 Agent 运行在容器中，将文件系统挂载为只读，白名单目录单独可写，这样即便代码漏洞被绕过，系统层也还有一道防线。

## 总结

给 Agent 脚本加目录白名单，本质是用“默认拒绝”的思想代替“默认允许”，它不能百分百防住所有高级攻击，但能在绝大多数无害化、误操作、简单路径穿越的场景中兜底。工程实现上，十几行代码就能把核心的路径检查嵌入到工具栈中，配合`realpath`与分隔符补丁，足以覆盖常见坑。

如果你的 OpenClaw Tool 或 MCP 自定义脚本已经准备对外暴露，先别急着上架——花 20 分钟装好这个文件访问护栏，比事后补救的代价小得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/837bf20860d32fe6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/41c7610dd5d357f4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/c2400f657df0020d.png)

