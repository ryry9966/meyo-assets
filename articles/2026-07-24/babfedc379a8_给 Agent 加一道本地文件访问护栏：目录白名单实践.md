---
title: 给 Agent 加一道本地文件访问护栏：目录白名单实践
feedId: 30316
source: 综合讨论
publishedAt: 2026-07-24
---

## 为什么需要这道护栏

Agent 与自动化脚本越来越频繁地操作本地文件系统：读写配置、导出中间产物、缓存模型权重、生成报告。如果放任 Agent 拥有完整的文件读写能力，一个被注入的提示词、一个拼写错误的目标路径，或者模型幻觉出的文件名，都可能演变成安全事件——比如覆盖系统文件、遍历敏感目录、泄露 `/etc/passwd` 或 `~/.ssh`。

把文件访问权限收紧到“仅允许少数受信目录”，是低成本、高收益的工程化护栏。本文以 Python 运行环境为例，展示如何给一个 Agent 或 MCP 工具函数增加目录白名单校验，并在 OpenClaw 这类框架的插件/工具层落地。

## 问题场景

假设你给 Agent 注册了一个工具函数 `write_file(path, content)`，让它可以写文件。理想情况下，Agent 只会向 `/workspace/output/` 这类约定目录写入。但提示词边界很难绝对防御，Agent 可能会试图写入 `/etc/cron.d/evil` 或 `~/.bashrc`。即使使用了“禁止写入系统目录”的提示词约束，依然存在绕过风险。

更隐蔽的路径绕过手段包括：
- 符号链接：`/workspace/output/link -> /etc/passwd`
- 相对路径：`../../../../etc/passwd`
- 裸路径与 `~` 展开：`~/../../etc/`
- 路径规范化差异：双斜杠、尾随空格、`.` 与 `..` 混合

只靠字符串前缀匹配（如 `startswith('/safe/dir')`）基本挡不住这些情况。

## 做法：一个可复用的路径白名单装饰器

核心思路是：**在真正的文件 I/O 之前，对路径做一次“解析→规范化→检查”**，确保最终要操作的真实路径落在白名单目录内。

下面给出一个可直接用于工具函数的装饰器示例（依赖 `pathlib`，无额外第三方库）：

```python
import os
from pathlib import Path
from functools import wraps
from typing import List, Union

def path_guard(allowed_dirs: List[Union[str, Path]]):
    """装饰器：仅允许操作白名单目录内的路径"""
    allowed_dirs = [Path(d).resolve(strict=False) for d in allowed_dirs]

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 约定：第一个参数或 key='path' 的入参是文件路径
            path_arg = args[0] if args else kwargs.get('path')
            if path_arg is None:
                raise ValueError("No path argument found for guard")

            target = Path(os.path.expanduser(path_arg)).resolve(strict=False)

            # 必须落在某一个允许目录内
            if not any(
                str(target).startswith(str(allow)) + os.sep or target == allow
                for allow in allowed_dirs
            ):
                raise PermissionError(
                    f"Path {target} is not allowed. Allowed: {allowed_dirs}"
                )
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

应用到工具函数：

```python
@path_guard(allowed_dirs=['/workspace/output', '/tmp/agent-safe'])
def write_file(path: str, content: str):
    Path(path).write_text(content)
```

在 OpenClaw 的工具注册环节，只需给涉及文件读写的函数加上该装饰器即可。如果工具函数较多，也可以抽象为一个 `safe_open` 上下文管理器，在 `open()` 调用前统一检查。

## 关键踩坑点

即使逻辑看起来简单，工程落地时还是踩过几个坑。

**1. `resolve()` 对不存在的路径无效**

`Path.resolve(strict=False)` 对于不存在的路径，会尽可能解析已存在的父目录部分，但最终路径仍保留不存在的文件名。若攻击者传入 `/workspace/output/../../etc/passwd`，此时 `/workspace/output` 已存在，`resolve` 仍能正确消除 `..`，得到 `/etc/passwd`，可以防御。但如果整个父目录都不存在，解析结果可能不按预想工作。建议先确保白名单目录已存在，或对父目录逐层 `resolve`。

**2. 符号链接竞态条件**

文件系统在检查时和实际写入时可能有变化（TOCTOU）。比如白名单内的某个子目录在检查后被替换为指向 `/etc` 的符号链接。缓解方法：
- 在操作前用 `os.lstat()` 判断是否为符号链接，禁止操作符号链接路径（除非明确需要）。
- 使用 `openat` + `O_NOFOLLOW` 等系统调用级标志（需 ctypes 或 py 3.11+ 的 `os.open` 的 `dir_fd`），但这会提高复杂度。
- 多数场景下，Agent 运行在单用户低权限容器内，竞态风险已大幅降低。

**3. `os.path.expanduser` 的陷阱**

`expanduser` 依赖 `HOME` 环境变量，如果 Agent 运行环境中 `HOME` 被篡改，可能被引导到非预期目录。更稳的做法是显式指定一个已知的 `HOME` 基准，或对 `~` 开头的路径直接拒绝，只允许绝对路径。

**4. Windows 兼容性**

如果你的 Agent 服务可能运行在 Windows 上，注意盘符、反斜杠、大小写不敏感等问题。`pathlib` 的 `resolve()` 在 Windows 上同样可用，但大小写判断需要额外小心。

## 可复用建议

根据实际部署经验，有一些可落地的模式：

- **区分读写权限**：为读取操作设置更宽松的白名单（如多个共享只读数据目录），为写入操作设置更严格的白名单，通常是 1~2 个目录。
- **监控拒绝日志**：每次 `PermissionError` 都记录完整的调用栈、原始路径和解析后路径，方便发现幻觉路径或攻击尝试。
- **默认拒绝，显式放行**：在 Agent 工具的 `open` 封装里，如果未配置白名单，直接抛出异常，防止因忘记加装饰器而暴露全盘。
- **结合容器/沙箱**：目录白名单是应用层护栏，应叠加操作系统级别的能力（如 Docker 的 `--read-only`、`--tmpfs`、`seccomp`），纵深防御。
- **给白名单目录加一个“安全根”**：例如 `/workspace/agent-sandbox/`，所有工具读写都以此为根，然后在里面按需挂载或链接必要数据。

## 总结

目录白名单是一种简单有效的文件访问控制手段，实现成本极低却能把 Agent 误操作的风险半径从一个文件系统缩小到几个可控目录。核心在于：解析并规范化真实路径，严格前缀匹配，并处理符号链接和相对路径绕过。配合日志审计与最小权限原则，它可以成为 Agent 安全基线的标准配置。

在 OpenClaw 的工具生态中，把这道护栏做到插件模板里，能让后续所有自定义工具天然带上文件访问约束——既不过度限制功能，又显著降低了自动化脚本“越界”的概率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/a5b4287f0fd54406.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/f7008c82761b9dda.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/39b3b853895bcc99.png)

