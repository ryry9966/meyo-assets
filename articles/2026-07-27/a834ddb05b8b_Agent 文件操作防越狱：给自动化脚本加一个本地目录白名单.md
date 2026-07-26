---
title: Agent 文件操作防越狱：给自动化脚本加一个本地目录白名单
feedId: 30624
source: 综合讨论
publishedAt: 2026-07-27
---

## 为什么需要这道“护栏”

在基于 OpenClaw / MCP 的 Agent 实践中，我们越来越依赖模型直接调用文件工具——读配置、写日志、清理临时目录，甚至批量重命名。一个典型的工具注册可能是：

```python
@tool
def write_file(path: str, content: str):
    with open(path, 'w') as f:
        f.write(content)
```

这种不加限制的能力非常危险。一个因为幻觉或 prompt 注入产生的路径（比如 `../../etc/crontab`）就能导致生产事故。我们需要在不牺牲工具灵活性的前提下，把文件操作严格限制在预设的“本地目录白名单”内，就像给自动化脚本加一道物理隔离的护栏。

## 问题拆解

目标很明确：所有通过 Agent 工具发起的文件读写、列表、删除、移动操作，其目标路径必须落在白名单目录内（例如 `/app/workspace`、`/data/output`）。拒绝任何试图逃逸到外部的请求。

看似简单，落地时至少要解决三个技术点：

1. **路径规范化**：处理 `../`、多余的 `/`、以及工作目录不确定带来的相对路径偏差。
2. **符号链接穿透**：即使是 `/app/workspace/link` 也可能指向 `/etc`，白名单要能识别这一层。
3. **跨操作一致性**：不仅 `write_file`，`move`、`copy` 的源和目标都需要受检。

## 实现步骤（以 Python 工具服务器为例）

### 1. 定义白名单配置

从环境变量或配置文件读入，避免硬编码：

```python
import os
WHITELIST_DIRS = [
    os.path.realpath(p) for p in os.getenv("FS_WHITELIST", "/app/workspace").split(":")
]
```

这里提前做一次 `realpath`，方便后续统一比较。

### 2. 编写路径安全校验函数

这是核心。思路：无论如何接收路径，都先基于一个安全的 `base_dir` 解析成绝对路径，再 `realpath` 去掉所有符号链接，最后判断是否以白名单目录为前缀。

```python
def safe_resolve(user_path: str, base_dir: str = None) -> str:
    if base_dir is None:
        base_dir = WHITELIST_DIRS[0]   # 建议显式传入
    # 将 user_path 相对于 base_dir 拼接，防止相对路径逃逸
    raw = os.path.join(base_dir, user_path) if not os.path.isabs(user_path) else user_path
    real = os.path.realpath(raw)
    if not any(real.startswith(wd + os.sep) for wd in WHITELIST_DIRS):
        raise PermissionError(f"Access denied for path {user_path} (resolved: {real})")
    return real
```

注意：如果用户传入绝对路径（如 `/etc/passwd`），这里也直接交给 `realpath` 处理，只要它不在白名单内，就会被拦下。更安全的做法是统一要求所有操作基于 `base_dir`，强制将绝对路径也视为错误。

### 3. 为每个文件工具挂载检查

```python
@tool
def read_file(path: str) -> str:
    safe_path = safe_resolve(path, base_dir="/app/workspace")
    with open(safe_path, 'r') as f:
        return f.read()

@tool
def move_file(src: str, dst: str):
    safe_src = safe_resolve(src, base_dir="/app/workspace")
    safe_dst = safe_resolve(dst, base_dir="/app/workspace")
    os.rename(safe_src, safe_dst)
```

列表工具（`list_files`）也适用，甚至可以进一步限制列举深度，但至少目录本身需合法。

## 踩坑实录

1. **工作目录漂移**：很多工具会依赖 Python 进程的当前工作目录。如果 Agent 进程在不同时刻 `cwd` 发生变化，`safe_resolve` 里的 `os.path.join(base_dir, user_path)` 和隐式的 `os.getcwd()` 会产生混淆。务必在工具函数里显式传入 `base_dir`，并确保 `base_dir` 为绝对路径且不依赖 `cwd`。

2. **符号链接的时机**：必须用 `os.path.realpath` 而不是 `os.path.abspath`，后者不解析符号链接。一个看似无害的工作区软链接指向根目录，就能绕过检查。另外，如果在白名单目录内部创建指向外部的链接，Agent 能否操作？我的建议是在安全策略中也禁止工具创建或修改符号链接，或者额外检查链接指向。

3. **操作特性遗漏**：`copy`、`move` 必须同时检查源和目标，`delete` 也要检查以免误删白名单外的文件。如果你开放了 `glob` 类操作，展开后的每个路径都要走一次校验。

4. **Windows 兼容性**：路径分隔符和大小写问题。使用 `os.path.realpath` 并统一比较时使用 `os.path.normcase`，或将白名单目录也做 `normcase` 处理。但大部分生产环境仍是 Linux，把这点明确在文档里即可。

## 可复用建议

- **抽象成中间件**：如果使用 MCP/OpenClaw 的工具注册机制，可以写一个装饰器 `@require_safe_path`，内部自动从参数中提取 `path`、`src`、`dst` 等常见字段进行校验，减少重复代码。
- **配置化并审计**：白名单通过环境变量注入，拒绝访问时打印完整的原始路径、解析路径和调用栈，便于第一时间发现注入尝试或脚本 bug。
- **结合用户反馈**：被捕的路径可以让 Agent 感知到“操作被拒绝”，而不是静默失败。返回清晰的结构化错误，模型可以据此调整后续操作，提升交互体验。
- **定期演练**：用 `../../etc/passwd`、符号链接、绝对路径等用例编写测试，保证每次工具升级后安全逻辑不被破坏。

## 总结

这层白名单护栏本质上是“最小权限”在文件系统上的落地。它不需要依赖沙箱或容器，实现成本很低，但在你让 Agent 获得了执行脚本、读写文件的能力后，能有效阻止最粗粒度的意外破坏。配合合理的目录结构与权限分离，你会得到一个既灵活又可控的自动化助手，而不是一台随时可能删库的定时炸弹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/5b5edf5cfe8f0eb2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/881cf61990a82891.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/252fee63ee374d28.png)

