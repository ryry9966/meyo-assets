---
title: 给 Agent 脚本套上缰绳：本地文件访问白名单实践
feedId: 28983
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景

在自建 Agent 或基于 OpenClaw 拓展工具链时，我们经常让脚本直接读写本地文件：生成报告、归档日志、处理用户上传的文档。LLM 决定调用哪个工具、传入什么参数，整个过程人手参与的环节极少。这让自动化程度变高，也带来了一个朴素却致命的问题——Agent 现在可以“不小心”删掉你的 SSH 密钥，或者在 `~/Documents` 里遍历所有私人文件并打包外传。

把 Agent 整个放进 Docker 容器是一种办法，但太重，而且很多临时任务产生的文件本来就需要落在宿主机目录里。更轻量的工程做法是在**脚本工具层**加一层白名单护栏：只允许 Agent 在预先划定的目录树内进行文件操作，对越界访问直接拦截。

## 问题拆解

文件访问护栏需要解决三个具体场景：

1. **直接路径越界**：Agent 传参 `/etc/passwd` 或 `../../.ssh/id_rsa`。
2. **符号链接绕过**：白名单目录内存在指向外部敏感文件的软链接，看似合法，实则会泄露数据。
3. **跨平台路径差异**：Windows 反斜杠、盘符、大小写不敏感等特性容易让校验逻辑失效。

下面基于 Python `pathlib` 给出一个在 OpenClaw 自定义工具中可直接落地的实现，其它语言思路一致。

## 做法：三步实现文件白名单校验

### 1. 定义白名单配置

用列表管理允许的根目录，最好通过环境变量或配置文件注入，避免硬编码：

```python
import os
from pathlib import Path

ALLOWED_ROOTS = [
    Path(os.environ.get("AGENT_WORK_DIR", "/var/agent/workspace")),
    Path("/tmp/agent-staging"),
]
READ_ONLY = True   # 是否需要写保护，可扩展为权限位
```

### 2. 编写路径校验函数

核心逻辑：规范化路径 → 解析符号链接 → 检查是否以任一白名单根为前缀。

```python
def is_path_allowed(target: str) -> bool:
    try:
        p = Path(target).expanduser().resolve()
    except (OSError, RuntimeError):
        # 路径无效或循环符号链接直接拒绝
        return False

    for root in ALLOWED_ROOTS:
        try:
            # 确保 root 也是绝对且已解析的
            root_resolved = root.resolve()
            # 使用 Path.is_relative_to （Python 3.9+）
            if p.is_relative_to(root_resolved):
                return True
        except OSError:
            continue
    return False
```

要点：
- `expanduser()` 把 `~` 替换为家目录绝对路径，防止变体绕过。
- `resolve()` 不仅规范化 `../`，还会**跟随并解析所有符号链接**，直接解除软链接跳板问题。
- `is_relative_to()` 是 Python 3.9 引入的，如果还在用老版本，可以用 `str(p).startswith(str(root_resolved) + os.sep)` 替代，但要注意处理短路径前缀匹配的错误。

### 3. 在工具函数入口集成

不用在每个工具函数里重复校验，写一个装饰器即可：

```python
import functools

def require_safe_path(arg_index=0):
    """装饰器：要求第 arg_index 个参数（通常是文件路径）通过白名单校验"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            path_arg = args[arg_index]
            if not is_path_allowed(path_arg):
                raise PermissionError(f"Access denied: {path_arg}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@require_safe_path(arg_index=0)
def read_file_content(filepath: str) -> str:
    with open(filepath) as f:
        return f.read()
```

对于有多路径参数的工具，可以在内部显式调用 `is_path_allowed`，并在拒绝时返回结构化的错误信息，方便 LLM 根据反馈调整行为，而不是直接 500。

## 踩坑记录

- **循环符号链接**：`Path.resolve()` 默认遇到循环链接会抛出 `RuntimeError`，我们直接捕获并拒绝，避免死循环。
- **挂载点与 overlay**：Docker/Kubernetes 环境里，白名单目录可能是宿主机 bind mount 进来的，`resolve()` 后路径可能变成容器视角的绝对路径，与白名单里配置的容器路径一致就不会有问题。但要保证配置与解析后的路径在同一个命名空间内。
- **已删除/不存在路径**：当 Agent 想要**创建**一个新文件时，路径还不存在，`resolve()` 会移除不存在的尾部组件。例如 `Path("/var/agent/new_dir/../new_file").resolve()` 会因为 `new_dir` 不存在而抛出 `FileNotFoundError`（在 strict=True 模式下）。解决方案：先对父目录做 resolve，再拼接文件名，或者使用 `resolve(strict=False)` 完全不抛异常。但 `strict=False` 可能放过部分异常情况，需要权衡。实践中我推荐**仅对存在的父目录部分 resolve**，用 `Path(target).parent.resolve() / Path(target).name` 的组合方式。
- **写操作的黑洞**：白名单只做了路径检查，但没有限制写操作本身。如果你的 Agent 只有“写临时文件”的需求，应该把工具定义为只写模式，或者额外限制只能写入特定子目录，并定期清理。否则 Agent 可能被诱导填满磁盘。
- **Windows 盘符与短路径**：若服务跑在 Windows 上，最好在 `is_path_allowed` 里强制将路径转为小写再做前缀匹配，同时要处理 `\\?\` 长路径前缀。但鉴于 OpenClaw 生态大多跑在 Linux 上，此坑优先级稍低。

## 可复用建议

- **封装为独立模块**：把上述校验逻辑连同装饰器做成一个 `agent_file_guard` 包，通过 pip 可安装，方便在多个项目里复用。还可以支持 “只读白名单” 和 “读写白名单” 分开配置。
- **与 MCP 工具定义联动**：若使用 OpenClaw 的 MCP 调用方式，可在 tool 的 description 里明确告诉 LLM “只能操作 /var/agent 下的文件”，减少无效越界调用。这相当于在 prompt 层和校验层双保险。
- **日志与告警**：每次拦截都应记录被拒绝的路径、时间、工具名，便于事后审计。如果短时间大量拒绝，可能是 prompt 质量不佳或者有人在试探，值得告警。
- **最小权限运行环境**：即使有白名单，也建议让 Agent 进程本身使用专门的系统用户，其家目录和其他目录默认无可读权限，形成纵深防御。

## 总结

本地目录白名单是 Agent 文件操作的“安全带”，实现成本极低，却能拦截绝大部分因 prompt 误导或模型幻觉导致的文件越权行为。核心就是**路径规范化 + 解析符号链接 + 前缀归属检查**，再配合装饰器或中间件将其无感植入工具层。落地时注意处理好不存在路径的 resolve 陷阱，并保持白名单配置的可注入性，它就足以成为你自建 Agent 基础设施里最可靠的默认防线之一。

---

## 配图

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/3b230f5335769d64.png)

