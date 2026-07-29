---
title: 避免自动化脚本“越狱”读写：为 Agent 文件操作挂载目录白名单
feedId: 30973
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：Agent 的文件操作正在变得“无边界”

在 OpenClaw、MCP 工具链或自定义 Agent 的实践中，经常会让 LLM 生成并执行一段脚本：数据处理、文件清理、批量重命名、导出报表等。这些脚本通常以子进程或内嵌 `exec` 的方式运行在宿主机上，几乎拥有当前用户的所有文件系统权限。

一旦 prompt 被污染、模型输出异常，或者需求本身隐含了“删除临时文件”这样的危险动作，可能出现以下情况：

- 误删项目外的重要文件（如 `~/.ssh` 或 `/etc` 下的配置）
- 读取到密钥文件、数据库配置
- 脚本利用相对路径跳出预期目录

典型的防御手段是容器沙箱或 `chroot`，但对于轻量级 agent 插件来说太重，而且会打断本地调试体验。因此，一个成本更低、可渐进接入的方案是：**在脚本执行层加一层本地目录白名单检查**，拦截所有越界文件访问。

这篇文章以 Python 环境为例，给出一个可直接复用的实现思路，同时说明在实际工程中容易忽略的坑。

## 问题拆解：我们需要拦截什么

目标：给定一个文件夹列表（白名单），所有 agent 生成的脚本只能读写这些路径下的文件，其他操作直接抛出异常或返回权限错误。

在 Python 中，文件操作主要通过：
- 内置函数 `open()`
- `os` 模块：`remove`, `rename`, `listdir`, `mkdir` 等
- `shutil`：`copy`, `rmtree`

如果只是简单地在脚本入口处拦截，需要在执行脚本前完成两件事：
1. **解析路径**：规范化绝对路径，避免相对路径和符号链接绕过
2. **检查归属**：判断最终路径是否位于白名单目录之下

在实践中，“脚本”往往是一段字符串代码，通过 `exec()` 或 `subprocess` 执行。如果控制的是 `exec()`，我们可以通过 monkey-patch 内建 `open` 和 `os` 函数来强制检查；如果是 `subprocess`，可以通过注入包装后的 Python 解释器或限制启动参数来约束，但更可控的方案仍是直接在 Python 进程中执行并 hook。

## 做法步骤：构建可复用的白名单执行器

以下实现适用于通过 Python 动态执行脚本的 Agent（例如 OpenClaw 中的自定义工具函数）。假设白名单目录为 `/data/sandbox` 和 `/tmp/agent_work`。

### 1. 路径规范化与检查函数

```python
import os

SAFE_ROOTS = [
    os.path.realpath("/data/sandbox"),
    os.path.realpath("/tmp/agent_work"),
]

def is_safe_path(path: str) -> bool:
    # 解析绝对路径，消除符号链接和 ../
    real_path = os.path.realpath(path)
    for root in SAFE_ROOTS:
        # 确保 real_path 以 root + os.sep 开头，或完全相等
        if real_path == root or real_path.startswith(root + os.sep):
            return True
    return False
```

注意 `os.path.realpath` 会跟随符号链接，避免 `/data/sandbox/link_to_etc` 指向白名单外的情况。白名单目录本身已经解析过。

### 2. 安全版 open 与原函数替换

```python
import builtins

original_open = builtins.open

def safe_open(file, mode='r', *args, **kwargs):
    if not is_safe_path(file):
        raise PermissionError(f"Access denied: {file}")
    return original_open(file, mode, *args, **kwargs)
```

同样处理 `os.remove`、`shutil.rmtree` 等。可以通过一个环境管理器来临时替换：

```python
import os
import shutil
from contextlib import contextmanager

@contextmanager
def filesystem_guard():
    # 保存原函数
    _open = builtins.open
    _os_remove = os.remove
    _os_rename = os.rename
    _shutil_copy = shutil.copy
    _shutil_rmtree = shutil.rmtree

    def guarded_remove(path, *args, **kwargs):
        if not is_safe_path(path):
            raise PermissionError(f"Remove denied: {path}")
        return _os_remove(path, *args, **kwargs)

    # 类似定义 guarded_rename 等...

    builtins.open = safe_open
    os.remove = guarded_remove
    os.rename = guarded_rename
    shutil.copy = guarded_copy
    shutil.rmtree = guarded_rmtree

    try:
        yield
    finally:
        # 还原
        builtins.open = _open
        os.remove = _os_remove
        os.rename = _os_rename
        shutil.copy = _shutil_copy
        shutil.rmtree = _shutil_rmtree
```

### 3. 执行 agent 脚本

```python
script = """
import os
with open('/etc/passwd') as f:
    print(f.read())
"""

with filesystem_guard():
    exec(script)   # 会抛出 PermissionError
```

而在正常的白名单内操作则通过：

```python
script_safe = """
with open('/data/sandbox/output.txt', 'w') as f:
    f.write('hello')
"""
with filesystem_guard():
    exec(script_safe)  # 正常执行
```

这构成了一个基础的“文件访问护栏”。

## 踩坑点

### 符号链接与挂载点绕过
光用 `os.path.abspath` 不够，务必使用 `os.path.realpath`。另外如果白名单目录本身是挂载点（例如 `/mnt/data`），要确认解析后的路径不会因为挂载变化而产生偏差。建议在启动时固化 `SAFE_ROOTS` 并拒绝在运行期间变更白名单。

### 路径分隔符与 Windows
如果 Agent 可能部署在 Windows 上，`os.sep` 要用于前缀检查；另外路径大小写不敏感可能带来漏判。建议统一转为 `os.path.normcase` 再比较，或直接基于 `os.path.realpath` 比较，Windows 上它会返回规范路径。

### 漏网之鱼：通过文件描述符传递
脚本可能通过 `os.fdopen` 操作一个已经打开的文件描述符。这种情况下我们很难在 hook 层拦截，因为路径已经在 `open` 时被检查过（如果 fd 来自一个安全文件）。如果 fd 是外部传入的，则防不住。因此需要确保 Agent 运行时不暴露危险文件描述符，或将整个进程的文件系统权限用操作系统能力（如 seccomp、AppArmor）收紧。

### 性能与嵌套调用
频繁的文件操作会多次校验路径，产生额外 `realpath` 系统调用。可以考虑加入 LRU 缓存，但必须小心目录被移动或符号链接变更的时效性问题。短期脚本场景直接实时解析即可。

### 模块级导入和 C 扩展
如果 Agent 脚本 import 了第三方库，这些库的内部文件读写（例如日志库写日志文件）同样会被拦截。如果日志目录不在白名单内，会导致异常。解决办法：在执行前提前配置日志输出到白名单目录，或为这些“系统行为”单独放行，但这会破坏安全模型的纯粹性。因此更务实的做法是：制定好严格的白名单目录结构，并让所有输出（包括临时文件）都写入其中。

## 可复用建议

1. **封装为 Agent Tool 的安全装饰器**  
   在 OpenClaw 插件或 MCP 工具中，将 `filesystem_guard` 做成一个可选装饰器：`@sandboxed(root_dirs=[...])`，让开发者显式声明。

2. **与 Prompt 结合显式失败信息**  
   不要只抛出冷冰冰的 `PermissionError`。可以 catch 异常后重试，返回给 LLM 一条明确的提示：“你的脚本试图访问白名单外的路径 `/etc/shadow`，已被拦截。请将所有文件操作限制在 `/data/sandbox` 目录内。”这样模型可以自我纠正。

3. **白名单列表从配置注入**  
   通过环境变量或 Agent 配置文件定义 `AGENT_SAFE_DIRS`，方便在不同部署环境修改，而不需要改代码。

4. **分层防御**  
   目录白名单是应用层护栏，不能替代系统级沙箱。如果安全等级要求高，配合 Docker 的 `--read-only` 加 tmpfs 或使用 bubblewrap 等工具，进一步限制文件系统可见范围。

5. **目录白名单的合理性**  
   避免将系统关键目录加入白名单，建议只给予项目数据目录和临时工作目录。Agent 需要读取的配置或模型文件，也应通过专门的只读映射或参数传递，而不是让脚本随意遍历。

## 总结

目录白名单是一种非常轻量的文件访问护栏，特别适合在自动化脚本引擎和 Agent 工具中快速落地。通过在 Python 层面 hook 文件操作函数，结合路径规范化和白名单检查，可以防止大部分意外越界读写。但工程师需要清楚它的局限：符号链接、文件描述符传递、C 扩展等可能绕过限制。把它作为纵深防御中的一环，与容器、权限控制、代码审查配合，才能让 Agent 的文件操作既灵活又可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/f9c42a757a21715d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/1aed583a9ea42b8e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/628dc94dc8f6b343.png)

