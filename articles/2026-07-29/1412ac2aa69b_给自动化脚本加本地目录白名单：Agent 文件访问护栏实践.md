---
title: 给自动化脚本加本地目录白名单：Agent 文件访问护栏实践
feedId: 30865
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在给 Agent 或 MCP 插件扩展文件读写能力时，最直接的做法是开放一个 `read_file` / `write_file` 工具，让模型可以操作任意路径。OpenClaw 生态里同样如此——工具函数如果写得过于宽松，Agent 一旦“想偏”或者遇到恶意 prompt，就可能访问到 `~/.ssh`、`/etc/passwd` 甚至工作目录之外的敏感文件。给文件访问加一道目录白名单，是成本最低、最工程化的防护手段之一。

这篇文章不会重新发明一个权限系统，而是给出一个能在 30 分钟内落地的方案，适合那些已经用 `subprocess`、`open()` 或 `pathlib` 跑自动化脚本、并希望在文件层面多一层防御的用户。

## 问题抽象

本质上我们要解决的是：**如何保证一个函数在执行任何文件 I/O 之前，操作的路径都必须落在开发者允许的目录集合内**。

常见场景：

- 你为 OpenClaw 写了一个插件，允许 Agent 读写 `./workspace` 下的项目文件，但不希望它碰到 `./config/secrets.yaml`。
- 你有一个批量重命名脚本，需要限制操作范围在 `/data/documents` 内，即使传入了 `../../etc/foo` 也要拒绝。
- MCP 工具要读写用户指定的目录，但符号链接可能会把真实路径引到白名单外。

在这些场景中，我们的核心需求是：**对任何待操作的路径，先规范化、再判断是否属于白名单目录子树**。

## 做法与步骤

以下以 Python 为例（因其在 Agent 工具中使用最多），但思路可以迁移到任何语言。

### 1. 定义白名单目录

选择一个或多个基础目录，全部存为 `pathlib.Path` 对象并调用 `resolve()` 得到绝对路径，这样后面比较时不会受符号链接和相对路径干扰。

```python
from pathlib import Path

ALLOWED_ROOTS = [
    Path("/home/user/project/workspace").resolve(),
    Path("/tmp/agent-sandbox").resolve(),
]
```

### 2. 编写校验函数

对于待操作的目标路径，同样 `resolve()`，然后用 Python 3.9+ 提供的 `Path.is_relative_to()` 判断是否是某个白名单目录的子孙：

```python
def is_path_allowed(target: Path) -> bool:
    try:
        resolved = target.resolve(strict=False)
    except (OSError, RuntimeError):
        return False  # 无法解析的路径一律拒绝

    return any(
        resolved.is_relative_to(root) for root in ALLOWED_ROOTS
    )
```

`strict=False` 允许路径不存在时也能解析（比如写文件前校验），避免因文件未创建就直接抛异常。

### 3. 在工具函数中插入检查

把原本直接 `open(path, "w")` 的地方改成先调 `is_path_allowed`：

```python
def safe_write_file(filepath: str, content: str) -> dict:
    target = Path(filepath)
    if not is_path_allowed(target):
        return {"error": f"Access denied: {filepath}"}
    target.write_text(content)
    return {"ok": True}
```

如果你有很多文件操作函数，可以用装饰器来减少重复：

```python
def require_allowed(func):
    def wrapper(filepath: str, *args, **kwargs):
        if not is_path_allowed(Path(filepath)):
            raise PermissionError(f"Path not allowed: {filepath}")
        return func(filepath, *args, **kwargs)
    return wrapper

@require_allowed
def read_file(path: str) -> str:
    return Path(path).read_text()
```

### 4. 处理符号链接绕过

`resolve()` 会跟随符号链接，所以如果用户故意在白名单目录里放一个指向 `/etc` 的软链接，`resolved` 会变成 `/etc/xxx`，`is_relative_to()` 会正确地返回 `False`。这正是我们想要的行为。如果你的场景希望**禁止在白名单内放置任何符号链接**，可以在创建文件时另外检查 `target.is_symlink()`；但对于大部分读写场景，上述逻辑已经足够。

### 5. 集成到 MCP 工具

如果你使用的是 MCP 标准工具（如 OpenClaw 的 MCP 插件），只需把 `server.tool()` 装饰的函数内部加上校验即可。对于 `read_file` / `write_file` 这类被模型频繁调用的动作，在这里做一道检查基本零感知，但安全收益明显。

## 踩坑点

- **相对路径与 `resolve()` 的依赖**：若进程的当前工作目录变化，`resolve()` 的结果也会变。要么统一在函数内部使用 `Path.cwd()` 作为基准，要么把所有传入的相对路径先以白名单根为基准进行拼接，例如 `root / user_path`，然后再解析。直接对用户传入的相对路径做 `resolve()` 会受 cwd 影响，可能偏离预期。
- **Windows 盘符与大小写**：在 Windows 上，`C:\project` 和 `c:\Project` 经过 `resolve()` 会统一，但直接字符串比较可能出错。`is_relative_to()` 在 Python 3.9+ 的 Windows 版已处理大小写，可以放心。
- **性能**：每次文件操作都做一次 `resolve()` 可能会产生文件系统调用。如果在一个循环中大量操作文件，可对同一路径缓存解析结果，或对白名单目录预先存储 `resolved` 版本。
- **拒绝服务**：如果白名单目录定义过大（比如根目录 `/`），那这个护栏就形同虚设。实际落地时最好把白名单范围控制到具体项目目录。

## 可复用建议

- **配置化**：把白名单放在 YAML/JSON 配置文件中，工具启动时加载，方便后续调整而不改代码。
- **日志记录**：在 `is_path_allowed()` 返回 `False` 时记录告警日志（包含时间、路径、调用者信息），方便事后审计。
- **测试**：写几个单元测试，覆盖正常路径、包含 `..` 的路径、通过符号链接逃逸的路径、不存在的路径，确保行为符合预期。
- **扩展到其他语言**：Node.js 可以使用 `path.resolve()` + `fs.realpath()` 后判断路径是否以白名单前缀开头；Go 可以用 `filepath.Abs()` + `filepath.EvalSymlinks()` 然后 `strings.HasPrefix()`。

## 总结

Agent 文件访问护栏不需要引入复杂的权限框架，一个目录白名单 + 路径规范化再加几个装饰器就能在工程上形成足够强的第一道防线。对自动化脚本、OpenClaw 插件、MCP 工具来说，30 行代码换来的安全感值得放进每一个读写能力的实现里。

如果你已经在用类似的方案，欢迎社区里一起补充边缘案例和更多语言的实现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/729b068d02e862e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/819925a2dcc5742c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/06c3211fbbf3deaa.png)

