---
title: 给 Agent 的自动化脚本加上本地目录白名单
feedId: 29960
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景：当 Agent 能读写文件时，护栏就变得必要

在 OpenClaw 以及类似的 Agent 工程实践中，我们常常会把文件系统作为工具（Tool）开放给 Agent，让它可以读取日志、写配置、生成报告，甚至批量处理本地文件。这个能力虽然实用，但一旦 Agent 能调用文件工具，安全问题就会立刻显现：

- 一段含有 prompt injection 的用户输入，可能诱使 Agent 去读 `/etc/passwd`、`~/.ssh/id_rsa` 或环境变量文件；
- 错误的推理或幻觉，可能把 `/etc/nginx/nginx.conf` 覆盖成空文件；
- 单个 file-write 工具如果没有路径限制，等同于给了 LLM 一发文件系统中的任意写漏洞。

因此，在将文件工具暴露给 Agent 之前，**必须加上一层文件访问护栏**。真正落地的做法不是让 Agent 自己去判断路径是否安全，而是在工具层的代码里硬性约束一个**本地目录白名单**，让 Agent 只能在受控的沙箱路径内行事。本文就围绕这一个点，给出可直接复用的步骤和真实踩坑记录。

## 问题拆解：目录白名单需要解决哪些攻击面

一个朴素的做法是：在工具函数的开头，检查传入的文件路径是否以 `/safe/workspace` 开头。但这种“字符串前缀匹配”在面对真实文件系统时几乎等于不设防：

- 路径遍历：传入 `/safe/workspace/../../etc/passwd`，前缀匹配会通过，但解析出来的真实路径已经跳出目录；
- 符号链接：允许目录下有一个指向 `/etc` 的 symlink，前缀匹配依然通过；
- 相对路径与工作目录的不一致：Agent 传 `config.yml`，当前工作目录如果被切换过，实际访问的可能是不受控的路径；
- Unicode 归一化问题：用全角 `/` 或特殊字符绕过的攻击在某些 OS 上仍可能生效。

所以，一个工程上可靠的目录白名单工具，要做到：**在任何情况下，最后被打开的文件，其真实绝对路径都必须在白名单目录之内。**

## 做法：用 `realpath` + 前缀检查构造安全文件句柄

我们以一个在 Python 工具中最小的可复用实现为例。工具函数负责提供文件的“读”或“写”能力，而安全校验逻辑独立为一个 `SafeFileAccess` 类。

**第一步：定义白名单并解析所有输入路径**

```python
import os
from pathlib import Path

class SafeFileAccess:
    def __init__(self, allowed_roots: list[str | Path]):
        # 预先解析白名单根目录，统一用 resolve() 得到绝对路径
        self.allowed_roots = [Path(root).resolve() for root in allowed_roots]

    def _resolve_safely(self, user_path: str | Path) -> Path:
        # 关键：realpath 会解析所有符号链接和 .. 并返回绝对路径
        return Path(os.path.realpath(Path(user_path)))

    def _is_within_roots(self, resolved: Path) -> bool:
        for root in self.allowed_roots:
            try:
                # 确保 resolved 在 root 之内，方法：比较父路径
                resolved.relative_to(root)
                return True
            except ValueError:
                continue
        return False

    def open(self, file_path: str | Path, mode: str = "r"):
        resolved = self._resolve_safely(file_path)
        if not self._is_within_roots(resolved):
            raise PermissionError(f"Access to {resolved} denied (not under allowed roots)")
        return open(resolved, mode)
```

上面的核心就两步：用 `os.path.realpath()` 拿到真实绝对路径；然后用 `Path.relative_to()` 检查前缀关系。**必须使用 `realpath` 而非 `resolve`**，因为 `resolve` 默认不处理 symlink（`strict=False` 时），而 `realpath` 会直接返回解析后的真正路径，确保没有符号链接绕道。

**第二步：在 Agent 工具中复用这个句柄**

假设 OpenClaw 中的工具定义如下：

```python
from typing import Annotated

class FileWriteTool:
    def __init__(self, safe_access: SafeFileAccess):
        self.safe_access = safe_access

    def run(self, file_path: Annotated[str, "文件路径"], content: str):
        with self.safe_access.open(file_path, mode="w") as f:
            f.write(content)
        return {"status": "ok"}
```

同样的 `FileReadTool`, `FileDeleteTool` 都可以共用同一个 `SafeFileAccess` 实例。白名单可以在启动时从环境变量注入：

```python
import os
allowed = os.getenv("AGENT_ALLOWED_PATHS", "./agent_workspace")
safe = SafeFileAccess(allowed.split(":"))
```

这样，即便未来 Agent 暴露出新的文件工具，只要都走 `safe.open()`，就不存在逃逸的可能。

## 踩坑点：三个容易被忽略的反向边界

**1. 不要相信 `os.path.abspath` 的前缀检查**

`abspath` 只是把相对路径拼接上当前工作目录，但不解析符号链接。如果白名单目录下的一个子目录是 symlink，那么 `abspath` 给出的路径看上去还在目录内，但它对应的真实文件系统位置可能已经飞出白名单。

正确做法永远是先拿到真实路径（`realpath`），再判定归属。

**2. 工作目录本身也可能被篡改**

如果 Agent 有执行 shell 命令的工具，就可能发生 `cd /tmp && rm -rf config.yml` 的情况。你传给 safe access 的相对路径会在当前工作目录下解析，而当前工作目录可能是 `/tmp`。因此，在解析相对路径之前，要么强制将相对路径拼接到固定的基路径（例如 `self.workspace_base / user_path`），要么直接禁止相对路径，只接受绝对路径或基于白名单根目录的相对路径。我的建议是：在安全层**统一基于白名单根目录解析相对路径**，对外暴露的 API 就不再依赖进程级的 `cwd`。

**3. 竞态条件：检查与使用的时间差**

`realpath` 检查完后，到 `open` 之前，文件系统可能已经变化（比如 symlink 被替换）。对 Agent 场景，通常不会刻意利用 TOCTOU，但如果你的 Agent 运行在多进程或共享目录下，这一窗口还是存在。短期最务实的缓解是在 `open` 时使用 `O_NOFOLLOW` 标志 + `os.open`（对 Linux），从而禁止跟随最后一层符号链接，并在打开后再做一次路径检查。对于非高安全诉求的业务，这一层可以暂时留作已知风险。

## 可复用建议：将护栏沉淀为组织级工具基类

在整个 Agent 工具生态里，文件护栏不是一锤子买卖。我会建议这样演进：

- **抽象 `SafeFileAccess` 为内部 SDK**：无论以后用 LangChain、OpenClaw 还是自研 Agent 框架，所有文件操作工具都共享并复用同一组白名单检查。
- **日志与告警**：当拒绝访问时，记录用户传入的原始路径、解析后的真实路径，并通过日志系统发出警告，便于发现潜在的 prompt 攻击或误用。
- **与系统权限配合**：以专用 Linux 用户运行 Agent 进程，该用户仅对白名单目录有 r/w 权限。即使代码层屏障被突破，系统 DAC 也是第二道防线。
- **容器化兜底**：如果 Agent 需要更严格的隔离，直接使用 Docker 的卷挂载，且指定 `:ro` 或仅挂载白名单目录，其余部分完全不可见。

## 总结

给 Agent 的自动化脚本加上本地目录白名单，是投入成本很低但安全收益很高的一项工程护栏。实现思路并不复杂：用真实路径解析消灭符号链接和路径遍历，用前缀判定保证文件一定落在白名单根目录内，再把这一逻辑固化成所有文件工具的必经入口。踩过的坑主要集中在函数选用（realpath vs abspath）、相对路径解析基准以及极端的 TOCTOU 场景。最终目标，是让每次文件操作都经过同一道闸口，而不是靠模型自己去“自觉”。

在 Agent 功能越来越像“自动执行体”的过程中，这种显式、强制、可审计的护栏，才是安全感的来源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/ff89666271c956e9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/62cfdb2797f364d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/2c8b4cb82ec7d12b.png)

