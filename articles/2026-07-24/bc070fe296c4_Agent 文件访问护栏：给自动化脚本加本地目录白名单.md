---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30252
source: 综合讨论
publishedAt: 2026-07-24
---

# Agent 文件访问护栏：给自动化脚本加本地目录白名单

## 为什么需要目录白名单

当 Agent 通过 MCP server 或 Function Calling 获得文件系统能力时，默认的“它可以访问任何路径”很快会变成安全隐患。无论是本地插件执行批量文件替换，还是 Agent 帮用户整理下载目录，如果传入一个 `../../.ssh/id_rsa`，灾难就发生了。

在自动化实践里，很多开发者会给 LLM 一个写死的 `work_dir`，然后相信模型不会越界。这种假设在单步操作中勉强成立，但 Agent 一旦具备多步推理和工具链调用，路径就可能来自上一步输出、用户输入、外部 API 响应，甚至被注入。**最可靠的护栏不是 prompt，而是在工具接口层做强校验。**

本文给出一个可直接落地到 OpenClaw、MCP server 或自定义插件的目录白名单方案，核心思路：**所有文件访问先过一层路径规范化器，只放行落在白名单目录内的真实路径。**

## 问题拆解

给文件操作加白名单，不是简单比对前缀：

- 相对路径 vs 绝对路径：`data/../config` 不能靠字符串前缀判断。
- 符号链接：`/safe/dir/link -> /etc` 需要解析后判断。
- 路径规范化在不同 OS 上的差异：Windows 的盘符、大小写、反斜杠；Linux 的 trailing slash。
- 多白名单目录：可能有 `./workspace` 和 `./outputs`，需要统一解析为绝对路径再比较。

所以核心逻辑应该是：
1. 将白名单目录列表全部转为规范化的绝对路径。
2. 对每个传入的文件路径，做 `realpath()`(解析符号链接)后，检查它是否以某个白名单路径为前缀。
3. 拒绝任何解析失败路径（`FileNotFoundError` 等）或未命中白名单的路径。

## 实现步骤

下面以 Python 为例，MCP server 或插件通常用 asyncio，这里用同步函数示意，异步场景直接包一层即可。

### 1. 定义白名单并规范化

```python
import os
from pathlib import Path, PureWindowsPath

class PathGuard:
    def __init__(self, allowed_dirs: list[str]):
        self.allowed = set()
        for d in allowed_dirs:
            # 转绝对路径并解析
            abs_dir = Path(d).expanduser().resolve(strict=False)
            if not abs_dir.exists():
                raise ValueError(f"白名单目录不存在: {abs_dir}")
            self.allowed.add(abs_dir)

    def is_allowed(self, user_path: str) -> Path:
        # expanduser 处理 ~，resolve 解析 .. 和符号链接
        resolved = Path(user_path).expanduser().resolve(strict=False)
        try:
            resolved = resolved.resolve(strict=True)
        except FileNotFoundError:
            # 对还不存在的写入路径，尝试解析父目录
            parent = resolved.parent
            if not parent.exists():
                return None
            parent = parent.resolve(strict=True)
            if not any(parent == a or a in parent.parents for a in self.allowed):
                return None
            # 检查最终拼接后的路径是否在允许树内（通过 commonpath）
            # 这里采用父目录通过则放行，但需再比对一次
        # 精确检查前缀
        for base in self.allowed:
            try:
                resolved.relative_to(base)
                return resolved
            except ValueError:
                continue
        return None
```

注意 `resolve(strict=True)` 会抛异常如果路径不存在。对“新建文件”的场景，我们可以用另一个策略：先解析父目录，确认父目录在白名单内，再拼接文件名，并验证最终绝对路径是否也在白名单内（防止 `../../` 穿越）。

一个更健壮的“允许新文件写入”版本：

```python
def resolve_safe_path(self, user_path: str) -> Path:
    raw = Path(user_path).expanduser()
    # 如果用户给了绝对路径，直接拼接会破坏白名单；不允许绝对路径
    if raw.is_absolute():
        raise PermissionError("只接受相对路径")
    # 拼接到一个白名单根下进行解析
    for base in self.allowed:
        candidate = (base / raw).resolve(strict=False)
        try:
            candidate = candidate.resolve(strict=True)
        except FileNotFoundError:
            # 父目录必须存在且在允许范围内
            if not candidate.parent.exists():
                continue
            candidate_parent = candidate.parent.resolve(strict=True)
            # 检查父目录前缀
            if any(candidate_parent == a or a in candidate_parent.parents for a in self.allowed):
                return candidate  # 父目录合法则信任，但需进一步检查前缀，这里简化
    raise PermissionError("路径不在白名单内")
```

这个版本强制传入相对路径，仅以白名单为根进行拼接解析，有效阻止了 `../../etc` 这种穿越。

### 2. 集成到工具调用层

在 MCP tool 或 Agent 插件中，所有读写文件的操作都通过一个统一的 `safe_path` 函数：

```python
def read_file(relative_path: str):
    abs_path = guard.resolve_safe_path(relative_path)
    return abs_path.read_text()

def write_file(relative_path: str, content: str):
    abs_path = guard.resolve_safe_path(relative_path)
    abs_path.parent.mkdir(parents=True, exist_ok=True)
    abs_path.write_text(content)
```

如果工具必须支持外部传入的绝对路径（不太建议），则在 `is_allowed` 中对绝对路径直接 resolve 后比对前缀即可，但要拒绝所有符号链接指向白名单之外的路径。

### 3. 配置化

白名单目录通过环境变量或配置文件注入：

```python
import os, json

config_path = os.getenv("AGENT_ALLOWED_DIRS_JSON", '["workspace", "outputs"]')
allowed = json.loads(config_path)
guard = PathGuard(allowed)
```

在 OpenClaw 的 skill 或 workflow 定义中，可以把白名单作为 tool 的参数，每次调用即用该工作流专属的目录，实现租户隔离。

## 踩坑记录

1. **符号链接绕过的双层解析**  
   `resolve()` 一次可以追符号链接，但如果白名单目录自身有符号链接组件，`resolve()` 后可能变成另一路径。务必用 `resolve()` 后再做 `relative_to` 比较，不要偷懒用 `samefile()` 检查——`samefile` 会认为 `/tmp/safe` 和 `/tmp/safe/..` 是同一文件而误判。

2. **Windows 大小写和盘符**  
   Windows 下 `Path.resolve()` 不会自动统一大小写，要防范 `C:\safe` 和 `c:\Safe\..\safe` 的不同。解决办法：在所有比对之前调用 `Path.resolve()` 后的 `casefold()` 做字符串比较，或一律 `os.path.normcase()`。但要注意 Python 的 `normcase` 在 Windows 上会转大写，在 Linux 上不变。

3. **新建文件场景的父目录验证**  
   对一个还不存在的文件 `output/reports/new.txt`，`resolve(strict=True)` 会抛 `FileNotFoundError`，这时必须向上回退。回退后要检查父目录是否合法，并重新拼接文件名后再次验证整体路径是否仍落在白名单内，否则攻击者可传入 `output/../../etc/passwd`，父目录 `output/../..` 解析后可能逃逸。

4. **多白名单的交集问题**  
   如果同时允许 `./data` 和 `./data/backup`，短的前缀会包含长的，不影响安全性。但不要反过来配置成前者是后者的子集，导致误拦截。建议校验配置时按最长路径优先排序，或者彻底规范化后使用 `relative_to` 逐个尝试，最后拒绝。

5. **不要依赖 user prompt 的路径过滤**  
   永远不要在工作流里对 LLM 说“禁止访问系统目录”——护栏必须是代码级的。模型可能被注入，或在上一步工具结果中包含恶意路径。

## 可复用建议

- **抽象为中间件**：在 OpenClaw 的 workflow 编排中，可以写一个 `with_file_guard(allowed_dirs)` 装饰器，对任何被 Agent 调用的文件操作自动检查。
- **审计日志**：白名单放行/拒绝都记录日志，带上工作流 ID 和请求 trace id，便于事后排查。
- **白名单表达支持通配符？** 不推荐。通配符可能引入类似 `./*/../../etc` 的逃逸，且跨平台行为不一致。如果确实需要，限定为固定的几级子目录模板（如 `workspace/{project}/`），并在运行时替换为具体值。
- **定期校验白名单目录权限**：用 `os.access` 检查读写权限是否符合预期，防止权限过宽（如 `777` 下的关键目录）。

## 总结

Agent 文件访问护栏应该简单、可审计、不能依赖模型自觉。目录白名单机制通过在工具层对每个文件路径进行规范化+前缀校验，能有效阻止绝大多数路径穿越和符号链接攻击。工程上，采用 `pathlib.Path.resolve()` + `relative_to` 的组合是跨平台相对稳妥的方案，再针对“文件尚不存在”的边界情况做父目录验证，就足以覆盖日常自动化任务。

安全不是一次性配置，而是要在每一次工具调用中重复检查。把这份逻辑做成插件或 skill 级别的组件，后续所有面向文件系统的 Agent 能力都能复用同一套护栏，远比每次手写校验代码可靠。

---

