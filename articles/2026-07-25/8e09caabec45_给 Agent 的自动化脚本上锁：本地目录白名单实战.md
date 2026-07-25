---
title: 给 Agent 的自动化脚本上锁：本地目录白名单实战
feedId: 30435
source: 综合讨论
publishedAt: 2026-07-25
---

## 为什么你的 Agent 需要文件访问护栏

在 OpenClaw 这类 Agent 框架里，我们经常让大模型调用本地脚本、MCP 工具或插件来操作文件——处理日志、导出报表、清理缓存。一旦 Agent 能执行 Shell 命令或运行 Python，风险窗口就打开了：一句 `rm -rf /`、一次意外的配置文件覆盖，或者把密钥文件读出来发给第三方，这类事故在自动化流水线中并不罕见。

容器化当然是强隔离，但不是所有场景都能用，也不是所有开发者都愿意为一个小工具拉个 Docker。更轻量的办法是：**在脚本运行前，给文件访问加上目录白名单**。这篇文章就是讲怎么在 Python 脚本和外部命令调用层面做这件事，不引入额外运行时，且可复用到 MCP 工具链中。

## 问题拆解：Agent 脚本文件访问的两条路径

Agent 操作文件通常通过两种方式：

1. **Python 代码本身** 使用 `open`、`os.remove`、`shutil.copy` 等函数。
2. **调用外部命令** 如 `cat /etc/passwd`、`tar -cf backup.tar /sensitive`。

要拦住这些操作，必须在这两个层面都做检查。核心思路是：

- 定义一个可配置的白名单目录列表（例如 `/home/agent/workspace`）。
- 每次文件操作前，将目标路径解析为**绝对路径**，并检查是否以白名单目录开头。
- 如果不在白名单内，拒绝执行并记录日志。

## 实现步骤：从装饰器到命令包装器

### 1. 路径校验核心函数

先写一个纯函数，负责路径规范化与检查。它需要处理相对路径、符号链接、`..` 穿越等常见绕过手段。

```python
import os
from pathlib import Path

WHITELIST_DIRS = [
    "/home/agent/workspace",
    "/tmp/agent_sandbox"
]

def is_path_allowed(target: str) -> bool:
    # 展开 ~ 和环境变量
    expanded = os.path.expandvars(os.path.expanduser(target))
    # 解析为绝对路径，同时解析符号链接
    resolved = Path(expanded).resolve()
    # 检查是否在任一台名单目录下
    for wdir in WHITELIST_DIRS:
        try:
            resolved.relative_to(Path(wdir).resolve())
            return True
        except ValueError:
            continue
    return False
```

这里使用 `Path.resolve()` 会同时处理符号链接和 `..`，避免 `../../etc/passwd` 这类绕过。

**踩坑点**：`Path.resolve()` 在路径不存在时会抛出异常，上面的写法没问题，因为只有在检查实际路径时才会解析；如果目标文件尚不存在但路径仍在白名单内，那么 `relative_to` 比较时仍然使用 `resolved` 代表的预期位置，但 `resolve()` 要求路径存在才能解析符号链接。稳妥的做法是：如果 `resolve()` 失败，先检查父目录是否存在，再解析父目录后拼接文件名。简化起见，生产代码需要额外处理这种边界情况。

### 2. 拦截 Python 文件操作

最直接的方式是提供一个安全的 `open` 替代函数，并强制在 Agent 脚本入口替换内置 `open`。但为了减少对现有代码的侵入，可以用上下文管理器或装饰器包装危险操作。

```python
import functools
import builtins

def safe_file_op(func):
    @functools.wraps(func)
    def wrapper(path, *args, **kwargs):
        if not is_path_allowed(str(path)):
            raise PermissionError(f"Access denied: {path}")
        return func(path, *args, **kwargs)
    return wrapper

# 替换内置 open
original_open = builtins.open
builtins.open = safe_file_op(original_open)
```

类似地，可以包装 `os.remove`、`shutil.copy` 等。但如果 Agent 调用了第三方库，那些库可能直接使用内置函数，所以这种 Monkey Patch 要在尽可能早的入口点执行（比如脚本最开始）。

**踩坑点**：Monkey Patch 会影响整个 Python 进程，包括你不想限制的代码（比如日志模块写临时日志）。可以改成在 Agent 任务执行前启用，执行后恢复，或使用更小的粒度，只对特定调用链使用装饰器。

### 3. 包装外部命令调用

Agent 很多时候用 `subprocess.run` 执行外部命令，我们需要检查命令参数中的文件路径。这部分无法做到完美，因为命令参数可能不以文件路径形式出现（比如 `echo "hello" > file` 通过 shell 重定向），但至少可以覆盖显式参数。

实现一个 `safe_subprocess_run`：

```python
import subprocess
import shlex

def safe_run(cmd, *args, **kwargs):
    # 仅做简单检查：将命令字符串拆开，对看起来像路径的参数做校验
    tokens = shlex.split(cmd) if isinstance(cmd, str) else cmd
    for token in tokens:
        # 忽略选项和标志
        if token.startswith('-'):
            continue
        # 如果 token 看起来像路径（包含 / 或 ~）
        if '/' in token or '~' in token or token == '.' or token == '..':
            if not is_path_allowed(token):
                raise PermissionError(f"Blocked path in command: {token}")
    return subprocess.run(cmd, *args, **kwargs)
```

注意：这无法拦截通过管道、重定向、环境变量等间接方式指定的文件，也无法防住编码混淆。但它能挡住大部分直接路径引用。

**踩坑点**：一些命令（如 `git -C /unlisted/path`）通过选项指定目录，`-C` 标志带路径但以 `-` 开头，会被上面规则跳过。需要根据具体命令定制解析逻辑，或者对每个已知的危险命令（`rm`、`cp`、`mv`）单独写更严格的校验。

### 4. 集成到 MCP 工具

如果你用 MCP 构建工具，可以在工具实现函数里直接调用 `is_path_allowed`。例如一个文件读取工具：

```python
async def handle_read_file(arguments):
    path = arguments.get("file_path")
    if not is_path_allowed(path):
        return {"error": "Access denied"}
    with open(path) as f:
        return {"content": f.read()}
```

这种做法比全局 Monkey Patch 更可控，也是推荐的方式。只要所有文件相关工具都加上这层检查，就能确保 Agent 无法越权。

### 5. 配合环境变量动态配置

将白名单路径放进环境变量 `AGENT_WHITELIST_DIRS`（逗号分隔），启动时读取，便于不同环境切换，不用改代码。

## 常见绕过与加固建议

- **符号链接**：已在 `resolve()` 中处理。
- **硬链接**：硬链接指向的文件即使路径不同，真实 inode 也在白名单目录外，但 `resolve()` 拿到的仍然是链接所在路径，而不是目标文件的真实路径。所以如果 Agent 有权限创建硬链接，可能绕过。解决：禁止 Agent 脚本创建硬链接，或使用文件系统 ACL。
- **相对路径与 chdir**：如果 Agent 先 `os.chdir` 到白名单外目录，然后 `open('file')` 就会读取那个目录的文件。我们的校验发生在文件操作时，会基于传入的路径（如 `file`）解析出绝对路径，发现它不在白名单内，因此拦截。但前提是路径校验器能拿到传入的原始字符串。如果用 Monkey Patch，`open('file')` 传入的 `path` 是 `'file'`，`is_path_allowed` 会基于当前工作目录解析，若 cwd 在白名单外，解析后的绝对路径就会暴露问题。所以同样需要检查解析后的绝对路径。这也是为什么 `is_path_allowed` 里面用了 `os.path.expanduser` 和 `Path.resolve()` ——它能将 `'file'` 转为 `/current/working/dir/file` 再判断。因此确实能阻挡 chdir 攻击。
- **通过进程替换**：例如 `vim -c ':!cat /etc/passwd'`，这种命令层面的逃逸需要禁止 Agent 运行交互式程序，或进一步限制命令白名单。

## 可复用的工程建议

- **做成独立的安全模块**：将 `is_path_allowed`、`safe_subprocess_run`、`safe_open` 放到一个 `agent_sandbox.py` 中，供所有脚本导入。
- **全量日志记录**：每当拒绝一次访问，记录完整堆栈和请求路径，方便排查。
- **与身份系统联动**：如果 Agent 在不同的用户下执行，可以将白名单与用户绑定。
- **只读挂载**：对于静态数据，可以用 `mount --bind -o ro` 的方式将目录只读暴露给 Agent，这比代码检查更底层，双保险。
- **测试你的护栏**：故意写一个尝试读取 `/etc/shadow` 的脚本，验证拦截是否生效。

## 总结

给 Agent 加文件访问护栏不是一蹴而就的安全方案，但在快速迭代的自动化场景里，它成本极低、见效快。用一个不到 30 行的 `is_path_allowed` 核心逻辑，结合对文件函数和子进程调用的包装，就能把 Agent 的破坏半径从整个文件系统缩小到几个指定目录。在还没有能力上完整容器沙箱的时候，这种轻量级白名单值得作为第一道防线。

**最后的提醒**：任何运行用户级代码的 Agent 都不应该以 root 身份运行。白名单是应用层限制，配合最小权限原则，才能最大程度降低风险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/3084452f7d923db7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/34174790e3eca785.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/b92496961b95a15e.png)

