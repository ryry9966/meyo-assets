---
title: Agent 文件访问护栏：用路径白名单锁死自动化脚本的读写边界
feedId: 30486
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：当 Automation 学会“乱翻”文件系统

在 Agent 与 MCP 工具链越来越普及的今天，让 LLM 驱动一个能读、能写、能调 shell 的自动化脚本已经稀松平常。但很多工程实践仍处于“全盘信任”状态：给脚本一个 `open()` 能力，它就能读取整个文件系统；给 `execute_command` 或 `run_shell`，就相当于交出 shell 权限。一旦 prompt 被注入、工具返回恶意内容，或者模型本身产生意料之外的路径组合，后果可能是关键配置文件被重写、SSH 密钥泄露、日志被污染。

这类问题不是安全圈的新鲜事——传统后端开发早就用 sandbox、chroot、capability drop 来限制进程行为。但在快速迭代的 Agent 工具链中，开发者往往更关心“能不能跑通”，而不是“能跑多少不该跑的东西”。**文件访问护栏**因此不是一个可选项，而是自动化脚本投入生产前的必修课。

## 问题定义：不是不让读写，而是划定“合法领土”

目标清晰但容易踩坑：我们想控制一个自动化脚本只允许在若干本地目录内进行文件操作（读、写、删除、列目录），超出即拒绝。这需要在不依赖容器/虚拟机等外部隔离手段的前提下，在**进程内**实现一个可靠的路径白名单校验。

关键约束：
- 自动化脚本可能以任何路径风格输入：绝对路径、相对路径、符号链接、`..` 回溯、多余的 `/./` 等。
- 校验必须对所有文件 API 调用生效，不能有遗忘的旁路。
- 性能开销要小，校验逻辑要简洁且可审计，避免引入复杂的权限模型。

## 做法：路径白名单校验器 + 工具包装层

### 1. 核心校验函数

实现一个 `is_path_allowed(target, allowed_roots)` 函数，它做三件事：

- 解析路径为**规范化的绝对路径**：`os.path.realpath(os.path.abspath(path))`
- 检查规范化后的路径是否以某个白名单根目录为前缀，并确保不是通过 `..` 等手段逃逸出去（使用 `os.path.commonpath` 比较）。
- 对 Windows 环境同时考虑盘符大小写、分隔符归一化。

Python 示例（精简版）：

```python
import os

def is_allowed_path(path: str, allowed_roots: list[str]) -> bool:
    # 避免空路径、None 等异常输入
    if not path or not isinstance(path, str):
        return False
    # 解析所有符号链接和相对路径，得到最终的真实绝对路径
    try:
        real_path = os.path.realpath(os.path.abspath(path))
    except (OSError, ValueError):
        return False
    # 对每个根目录做前缀比较，且要求不在根目录之上
    for root in allowed_roots:
        try:
            root_real = os.path.realpath(os.path.abspath(root))
            # commonpath 能防范类似 /allowed/../secret 绕过的残留风险
            if os.path.commonpath([real_path, root_real]) == root_real:
                return True
        except ValueError:
            continue
    return False
```

关键点：仅做字符串前缀匹配是不够的。`/data/allowed/../secret` 可能在解析前绕过了简单的字符串 `startswith`，但在经过 `os.path.realpath` 和 `commonpath` 双重校验后会显形。

### 2. 对文件操作 API 进行包装

不要直接向 Agent 工具暴露 `open()` 或 `os.remove()`。在工具定义层对每个文件操作进行路径检查，例如 MCP server 中的 `read_file` 工具实现：

```python
def safe_read_file(filename: str, allowed_roots: list[str]) -> str:
    if not is_allowed_path(filename, allowed_roots):
        raise PermissionError(f"Access denied: {filename}")
    with open(filename, "r", encoding="utf-8") as f:
        return f.read()
```

同理覆盖 `write_file`、`delete_file`、`list_directory`。**注意** `list_directory` 也要检查请求的目录是否在白名单中；同时返回的文件列表应只展示文件名，不意外暴露外部路径。

### 3. 注入到工具集

如果是基于 OpenClaw 或类似框架构建的自动化插件，可以在工具注册时统一注入路径验证装饰器：

```python
def path_guard(allowed_roots):
    def decorator(func):
        def wrapper(path, *args, **kwargs):
            if not is_allowed_path(path, allowed_roots):
                raise PermissionError(f"Path {path} not allowed")
            return func(path, *args, **kwargs)
        return wrapper
    return decorator
```

这样所有需要处理路径的工具只需加上 `@path_guard(ALLOWED_ROOTS)`，容易维护且不易遗漏。

## 踩坑记录：那些看起来无害的绕过手段

- **符号链接炸弹**：白名单目录内若存在指向外部敏感位置的符号链接，`realpath` 会将其解析到外部，因此校验必须使用 `realpath` 而不是 `abspath`。但这也带来一个新问题：如果你希望 Agent 能读取符号链接本身（不解析），则需区分语义。多数自动化场景下，解析链接后的真实路径才是安全风险评估的依据。
- **TOCTOU 竞态**：校验通过后到实际 `open()` 之间，文件系统可能发生变化（例如符号链接被替换）。这在本地单进程工具中概率较低，但如果追求更严格的安全性，可以在 `open()` 后再次用 `fstat`/`fd` 验证或采用 `O_NOFOLLOW` 打开。
- **Windows 路径陷阱**：`\\.\C:\data`、`\\?\C:\data` 等设备路径可能绕过普通规范化；建议直接禁止此类前缀，或统一用 `os.path.normpath` 处理。
- **白名单配置僵化**：一旦使用相对路径配置允许的根目录，必须保证在调用 `is_allowed_path` 前，这些根目录已经被绝对化和 `realpath` 处理，否则校验将不可靠。最佳实践是在启动时将白名单一次性解析并存储为规范化绝对路径列表。

## 可复用建议

1. **白名单即约定**：将 `ALLOWED_ROOTS` 作为工具或插件的显式配置项，放在环境变量或配置文件中，避免硬编码。
2. **最小权限原则**：为不同工具赋予不同白名单，例如日志写入仅允许 `./logs/`，数据读取仅允许 `./data/`，进一步缩小攻击面。
3. **统一路径入口**：编写一个内部 `FilePath` 封装类，所有文件工具都通过该类的类方法构造路径对象，构造时即完成校验，这样不会出现某条路径校验遗漏。
4. **日志与审计**：当路径被拒绝时，记录原始输入、解析后的路径以及白名单范围，用于事后排查误拒或攻击尝试。
5. **测试覆盖**：单元测试中至少包含：相对路径、符号链接指向外部、`..` 回溯、空字符串、超大路径、不存在的路径、以及多根目录之间的边界。

## 总结

文件访问护栏本质上是一种“防御性信任收缩”。它不解决所有安全问题，但能以极低的实现成本阻止最常见、破坏力最大的文件越权操作。在 Agent 和 MCP 工具爆发式落地的阶段，把护栏做成可复用的内置组件，远比每次亡羊补牢要划算得多。下一次当你给自动化脚本接入一个新的文件工具时，建议先问一句：**它是不是已经跑在白名单的安全区内？**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/64461c92df279160.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/1e6436fed95c5ada.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/267fb375c550b0fc.png)

