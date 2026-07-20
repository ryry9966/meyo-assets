---
title: 为Agent脚本加把锁：实施本地目录白名单的工程实践
feedId: 29864
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景：当 Agent 获得了文件系统的“自由”

无论是跑在 OpenClaw 里的自动化任务、MCP 工具调用，还是一段定期执行的插件脚本，只要 Agent 需要读写本地文件，权限往往被放任——脚本拿到什么路径就写什么路径，开发者也图省事，直接用 `open()` 随意操作。这在单机测试环境无伤大雅，但一旦 Agent 接到外部输入（用户指令、API 回调、LLM 生成的路径字符串），问题就来了：

- 恶意或异常的输入可能读取 `/etc/passwd`、`~/.ssh/id_rsa`
- 自动化整理文件的脚本可能误删系统目录
- 插件写得过于“灵活”，允许用户指定任意路径，等于开了后门

在工程实践里，给文件访问加上**本地目录白名单**，是最低成本、最高效的护栏。以下记录一套可直接复用的方案，包含实现、踩坑和模块化建议。

## 问题定义

一个典型场景：我为 OpenClaw 写了一个插件，功能是把 Agent 生成的 Markdown 渲染成 PDF，需要读写一个工作目录。插件函数签名可能是 `convert(file_path: str)`，如果用户传入 `../../.env`，插件就有可能把环境变量文件打包进 PDF 或触发读取报错。

我们的目标是：**让指定的代码片段只能读写预定义的目录集合，任何超出白名单的路径直接拒绝**。这个白名单由部署者配置，硬编码在运行时检查逻辑中。

## 步骤：从零实现路径护栏

### 1. 设计白名单解析与校验逻辑

使用 `pathlib.Path` 做路径解析，利用 `resolve()` 展开符号链接和相对路径，然后用 `is_relative_to()`（Python ≥3.9）判定路径是否在白名单目录之内。

```python
from pathlib import Path
from typing import Iterable, Union

class FileGuard:
    def __init__(self, allowed_dirs: Iterable[Union[str, Path]]):
        # 预先解析白名单，存储绝对路径
        self._allowed = [Path(d).resolve() for d in allowed_dirs]

    def guard(self, path: Union[str, Path]) -> Path:
        """检查路径是否安全，安全则返回 resolved Path，否则抛出异常"""
        resolved = Path(path).resolve()
        if not any(resolved.is_relative_to(d) for d in self._allowed):
            raise PermissionError(
                f"Access to '{resolved}' is not allowed. "
                f"Whitelist: {[str(d) for d in self._allowed]}"
            )
        return resolved
```

这里故意在返回语句中触发 `PermissionError`，不是为了真正做系统级权限控制，而是**在应用层快速失败**，避免后续的 `open()` 等操作接触到不该碰的文件。

### 2. 在脚本中集成

最简单的用法：在函数入口对所有路径参数调用 `guard`。

```python
guard = FileGuard(allowed_dirs=["/home/user/project/output", "/tmp/agent-sandbox"])

def convert_to_pdf(source: str, dest: str):
    safe_src = guard.guard(source)
    safe_dest = guard.guard(dest)
    # ... 后续使用 safe_src, safe_dest 进行文件操作
```

如果你有多个函数需要保护，可以把 `FileGuard` 实例封装为单例，或者用装饰器自动对参数中的路径字符串/Path 对象进行检查，减少重复代码。

### 3. 与 Agent 插件体系结合

如果你的 Agent 是 MCP 服务器提供的工具（如文件写入工具），那么在工具的实现函数里，同样注入 `FileGuard`。最佳实践是从环境变量中读取白名单目录列表，由部署者在 `docker-compose` 或系统服务文件中配置，不允许客户端（LLM 或用户）修改。

比如：

```python
import os

ALLOWED_DIRS = os.getenv("AGENT_ALLOWED_DIRS", "").split(",")
file_guard = FileGuard(ALLOWED_DIRS)
```

## 踩坑记录

实际落地时，几个细节容易翻车：

- **符号链接逃逸**  
  白名单目录是 `/data/safe`，但该目录下有个软链接指向 `/etc`。如果直接使用未解析的路径做检查，攻击者可以利用符号链接跳出限制。  
  解法：强制 `resolve()` 后再比对，并确保白名单目录本身也是解析后的路径。部分系统上 `/tmp` 可能是符号链接，需注意。

- **路径遍历攻击**  
  用户传入 `safe_dir/../../../etc/passwd`，如果不做解析，字符串检查会认为是在 `safe_dir` 下。  
  解法：同样是 `resolve()` 后判断。

- **跨平台与 Python 版本**  
  `is_relative_to()` 在 Python 3.9+ 才提供。如果你的运行环境是 3.8，需要自己实现：`resolved.is_relative_to(d)` 可替换为 `d in resolved.parents or resolved == d`。  
  Windows 盘符问题：白名单写 `C:\data`，而用户输入 `D:\proj`，`resolve()` 后跨盘符不会被误判为安全，但要注意 `Path` 对象在不同盘符下的行为。

- **性能敏感场景**  
  每次调用都 `resolve()` 可能带来额外 I/O（尤其循环内大量路径检查）。可对频繁访问的路径做缓存：`functools.lru_cache` 套在 `guard` 方法上，但需注意缓存失效策略。一般情况下调度频率不高，可忽略。

- **白名单范围过大**  
  `/` 或 `/home` 被误加为白名单，等于没有防护。启动时可加入合理性检查：比如如果白名单包含根目录或用户主目录的顶级，直接报错拒绝启动。

## 可复用建议

1. **封装为标准库**  
   将 `FileGuard` 连同装饰器、上下文管理器打包成一个轻量 pip 包，内部可依赖 `pathlib` 无额外引入。方便跨项目复用。

2. **整合到 OpenClaw 插件模板**  
   提交一个“文件安全工具”插件，提供 `FileGuard` 的实例化函数，以及读取环境变量的默认配置。其他插件直接 import 使用，形成社区共识的安全基线。

3. **测试用例覆盖**  
   为 `guard` 写参数化测试，覆盖场景：
   - 目录内普通文件（通过）
   - 兄弟目录（拒绝）
   - 父级目录（拒绝）
   - 软链接逃逸（拒绝）
   - 相对路径（转为绝对路径后通过）
   - 不存在的路径（仍需检查，因为可能创建新文件；这时 `resolve()` 会保留不存在的部分，需额外处理）

4. **与进程级沙箱互补**  
   `FileGuard` 是应用层护栏，不替代容器、chroot 等系统级隔离。实际部署仍建议配合 Docker 的卷挂载限制或 seccomp profile，形成纵深防御。

## 总结

文件访问白名单纯粹且有效，几行代码就能消除大部分路径注入风险。对于自动化的 Agent 脚本，做到“最小权限”并不需要复杂的 ACL 或内核模块，一个路径解析函数加白名单对比就够了。落地关键在于：**所有外部可触达的路径入口都加检查**、**始终解析符号链接**、**通过环境变量管理配置**。迁移到新项目几乎零成本，却能拦住不少意想不到的坑。

如果你正在维护一个会操作用户文件或系统路径的 Agent 插件，现在就可以给 `open()` 套上这个轻量的安全层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/1c6ad914393b1d23.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/082f2c049ba41fd9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/b34ad24ae15700ee.png)

