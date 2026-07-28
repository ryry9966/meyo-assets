---
title: 给 Agent 工具加一把锁：用本地目录白名单实现文件访问护栏
feedId: 30859
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 OpenClaw 这类支持工具调用的 Agent 框架中，让 Agent 读写本地文件几乎是一个刚需——生成报表要落盘，读取配置文件做决策，或者处理用户上传的数据集。但一旦把文件系统的能力暴露给大模型，风险也随之而来：没有约束的 prompt 注入可能诱导 Agent 去读取 `/etc/passwd`、删除用户目录，或者在 `/tmp` 下写一堆垃圾。

常见的“安全”手段是给 Agent 一个沙箱容器，但很多内部自动化场景跑在宿主机上，没条件做完整的容器隔离。这时工程上最务实的做法就是加一层文件访问护栏——通过本地目录白名单，把 Agent 的读写能力限制在明确允许的目录内。

## 问题

很多开发者习惯这样写工具函数：

```python
def read_file(path: str) -> str:
    with open(path, "r") as f:
        return f.read()
```

然后直接注册给 Agent。模型返回的 `path` 可能是 `../../.ssh/id_rsa`，甚至是用符号链接指向敏感位置的路径。即便你告诉 Agent“只能访问 /workspace”，大模型也可能被对抗性提示诱导越界。

护栏的核心诉求很清晰：**不论 Agent 输入的路径长什么样，最终解析出来的真实文件路径必须落在预设的白名单目录内**。

## 做法：基于路径规范化的白名单检查

### 1. 定义白名单并预先规范化

用 `pathlib` 的 `resolve()` 把白名单目录解析为无符号链接的绝对路径。注意这一步一定要在启动时做一次，不能每次请求都根据用户输入动态拼接。

```python
from pathlib import Path

ALLOWED_DIRS = [
    Path("/data/agent_workspace"),
    Path("/data/shared_readonly"),
]

RESOLVED_ALLOWED = [d.resolve() for d in ALLOWED_DIRS]
```

### 2. 编写路径安全校验函数

核心逻辑：将用户传入的路径也 `resolve()` 成绝对路径，然后检查它是不是以任意一个已解析的白名单目录开头。不要用字符串前缀比较，容易在边界上出问题（比如 `/data/agent_workspace_backup` 会被误认为合法）。推荐用 `Path.is_relative_to()`（Python 3.9+）或者比较路径的各部分。

```python
def is_allowed(path: str | Path) -> bool:
    target = Path(path).resolve()
    return any(target.is_relative_to(allowed) for allowed in RESOLVED_ALLOWED)
```

工具函数改造成这样：

```python
def safe_read_file(path: str) -> str:
    if not is_allowed(path):
        raise PermissionError(f"Access denied: {path}")
    return Path(path).read_text()
```

### 3. 覆盖写入、删除等操作

同样的校验可以封装成装饰器或 mixin，给所有涉及文件路径的工具调用加上。只要统一收口到一个安全校验点，后续新增工具时就不会漏。

```python
def guard_path(func):
    def wrapper(path: str, *args, **kwargs):
        if not is_allowed(path):
            raise PermissionError(f"Path not allowed: {path}")
        return func(path, *args, **kwargs)
    return wrapper

@guard_path
def write_file(path: str, content: str):
    Path(path).write_text(content)
```

## 踩坑点

1. **符号链接绕过**  
   白名单目录本身如果是一个符号链接（比如 `/data/agent_workspace -> /mnt/disk2/project`），那么直接 `resolve()` 用户输入后得到的真实路径会是 `/mnt/disk2/project/...`，它不以 `/data/agent_workspace` 开头，导致合法请求被拒。  
   **解法**：白名单目录必须提前 `resolve()`，在比较时用 `RESOLVED_ALLOWED`。如果业务需要保留符号链接形式，可以同时加入原始路径和解析后路径到白名单。

2. **相对路径与当前工作目录**  
   如果工具被调用时当前工作目录不确定，`resolve()` 会基于 `cwd` 把相对路径“补全”为绝对路径。这可能导致 `/tmp` 下的相对路径突然合法。  
   **建议**：工具入口强制只接受绝对路径，或者在 `resolve()` 之前先判断是否为绝对路径，不是则直接拒绝或补全到安全基路径。

3. **Windows 的盘符和长路径前缀**  
   在 Windows 上，`resolve()` 可能返回带 `\\?\` 的路径，盘符大小写也可能不一致。  
   **解法**：使用 `os.path.normcase()` 做标准化比较，或者统一用 `pathlib.Path.resolve()` 后比较 `parts`。

4. **TOCTOU 风险**  
   路径检查和实际操作之间存在时间窗口，但在这个场景下攻击面极小。如果环境真的高危，可以在操作前再做一次 `resolve` 检查，或直接用 `os.open` 加 `O_NOFOLLOW` 等 flag 防止符号链接逃逸，不过复杂度大幅增加，一般内部 Agent 场景不需要。

## 可复用建议

把这套检查封装成一个 `FileGuard` 类，提供 `read`/`write`/`list_dir` 等受限方法，整个项目的文件工具只通过它来访问磁盘。这样做有两个好处：

- 白名单配置集中管理，方便通过环境变量或配置文件动态设置；
- 以后需要增加审计日志、操作频率限制，只需要改这一个类。

最小化示例：

```python
class FileGuard:
    def __init__(self, allowed_dirs: list[Path]):
        self._allowed = [d.resolve() for d in allowed_dirs]

    def _check(self, path: Path):
        real = path.resolve()
        if not any(real.is_relative_to(d) for d in self._allowed):
            raise PermissionError(f"Access denied: {path}")
        return real

    def read_text(self, path: str) -> str:
        return self._check(Path(path)).read_text()

    def write_text(self, path: str, content: str):
        self._check(Path(path)).write_text(content)
```

在 Agent 工具注册时，只把这个 `FileGuard` 实例暴露出去，原始的文件操作不做注册。

## 总结

Agent 文件访问护栏并不是一个“可选的安全加分项”，而是直接决定你的自动化脚本能不能放心跑在生产环境里。目录白名单方案实现成本极低，只要在路径解析上多花一点心思，就能避免大部分因 prompt 注入或模型幻觉导致的误操作。对于 OpenClaw 或 MCP 工具开发者来说，建议从一开始就把文件操作收口到这一个安全层里，后面再去叠加速度限制、操作审计才会更顺畅。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/f3f145f732f21df2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/6586a0fb2aa9c183.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/f506aa6e0e4e8282.png)

