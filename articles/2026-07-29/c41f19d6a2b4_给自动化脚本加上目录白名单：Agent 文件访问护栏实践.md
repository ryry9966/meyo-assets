---
title: 给自动化脚本加上目录白名单：Agent 文件访问护栏实践
feedId: 30910
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景：为什么需要一个“护栏”

在 OpenClaw 这类 Agent 框架里，插件、MCP 工具或自动化脚本经常会直接与本地文件系统打交道。例如，一个自定义工具可能需要读取配置文件、输出处理结果、缓存中间数据。默认情况下，Python 的 `open()`、`os.remove()`、`shutil.rmtree()` 没有任何边界 —— 只要进程有权限，它可以操作整个文件系统。

实际的痛点出现在几个场景：

- **误删或误写**：脚本逻辑缺陷或参数注入导致操作了错误路径，比如把备份目录清空。
- **Agent 被提示注入**：如果 Agent 被诱导生成了包含 `../../../` 的路径，可以直接跳出工作区。
- **第三方插件不可信**：社区贡献的 MCP 工具可能存在审查盲区，却拥有和主进程相同的文件权限。

因此，我们需要一种轻量的“护栏”机制：**只允许脚本在预先声明的一组本地目录内读写，拒绝任何越界访问**。这不是安全沙箱，而是工程上最低成本的防御层。

## 问题拆解：要防什么，怎么防

我们需要解决的核心问题是：**任意给定的文件路径，是否落在白名单目录内？** 这与 Web 服务里的路径遍历防御非常相似，但脚本场景更加灵活，要处理相对路径、符号链接和不同操作系统。

一个看似简单的做法是检查路径前缀：

```python
if not filepath.startswith(allowed_dir):
    raise PermissionError
```

这很容易被绕过。比如 `allowed_dir = "/home/user/workspace"`，攻击者构造 `"/home/user/workspace/../secret.txt"`，`startswith` 会返回 `True`，但规范化后实际路径却是 `/home/user/secret.txt`。

因此必须**先规范化所有路径**。使用 `os.path.realpath()` 或 `pathlib.Path.resolve()` 得到不含符号链接和 `..` 的绝对路径，再判断是否以任一白名单目录为前缀。同时，目录本身也要规范化，并确保以路径分隔符结尾，避免前缀匹配错误（如 `/var/app` 匹配到 `/var/app_old`）。

## 实现步骤：一个可复用的工程化方案

在 Python 中，我倾向于用一个薄薄的文件访问层统一所有 I/O 操作。下面是核心实现要点，不依赖任何第三方库。

### 1. 定义白名单并规范化

```python
import os
from pathlib import Path

class FileGuard:
    def __init__(self, allowed_dirs: list[str]):
        # 全部转为规范的绝对路径，并确保以 os.sep 结尾
        self._allowed = set()
        for d in allowed_dirs:
            resolved = os.path.realpath(d)
            if not resolved.endswith(os.sep):
                resolved += os.sep
            self._allowed.add(resolved)
```

### 2. 路径校验函数

```python
    def _check(self, user_path: str) -> str:
        # 如果路径不存在，resolve 会报错，这里需要处理
        # 可以先对父目录 resolve 再拼接 basename，避免异常
        target = os.path.realpath(user_path)
        # 目录需要以分隔符结尾来精确匹配
        check = target if os.path.isdir(target) else target
        for base in self._allowed:
            if check.startswith(base) or check + os.sep == base:
                return target
        raise PermissionError(f"Access denied: {user_path}")
```

对于还不存在的路径（准备新建的文件），`os.path.realpath` 会因为路径不存在而保留原样，可能导致符号链接未解析。更严谨的做法是对已存在的父目录执行 `realpath`，再拼接文件名。这里可根据实际需求做增强。

### 3. 安全包装常用操作

```python
    def safe_open(self, path, mode='r', *args, **kwargs):
        safe_path = self._check(path)
        return open(safe_path, mode, *args, **kwargs)

    def safe_remove(self, path):
        safe_path = self._check(path)
        os.remove(safe_path)
    # ... 类似封装其他函数
```

### 4. 集成到 MCP 工具或 Agent 插件

在 Agent 的每个文件工具函数入口调用 `guard.safe_open` 替代原生 `open`。例如 MCP server 的某个工具：

```python
guard = FileGuard(["/app/workspace", "/tmp/agent_cache"])

@app.tool()
def read_file(filename: str) -> str:
    with guard.safe_open(filename, 'r') as f:
        return f.read()
```

如果工具是用户自定义的，推荐提供一个装饰器自动校验路径参数，减少重复代码。

## 实际踩坑点

**1. 符号链接突破**  
某次配置白名单时误把工作目录设置为一个软链接的真实父目录，而脚本使用软链接路径访问，`realpath` 解析后因为不包含白名单前缀而被拒绝。解决办法是在白名单中加入软链接指向的真实路径，并统一使用 `realpath` 比较。

**2. Windows 路径比较**  
Windows 盘符大小写、`\\` 和 `/` 混用。使用 `os.path.realpath` 会统一为带盘符的形式，但必须注意 `C:` 和 `c:` 是否一致。另外，`os.sep` 在对路径追加分隔符时要考虑跨平台。

**3. 性能陷阱**  
每次文件操作都调用 `realpath` 会产生 stat 系统调用，高并发场景下需要缓存已解析的路径白名单，并且对近期访问的合法路径做一个 TTL 缓存，减少重复解析。

**4. 相对路径与工作目录**  
脚本执行过程中可能 `os.chdir()` 改变当前目录，`realpath` 的相对路径基准就变了。建议在 `_check` 中先用 `os.path.abspath` 基于绝对 CWD 处理相对路径。

## 可复用建议

- **封装统一 I/O 层**：哪怕项目再小，也不要到处散布原生 `open`。一个 `FileGuard` 类成本极低，却能在未来扩展审计日志、配额管理。
- **白名单最小化**：只开放必要的目录，临时目录也最好限定在子目录，避免 `/tmp` 下其他进程的文件被误触。
- **加入审计**：在拒绝访问时打印完整调用栈和路径，方便排查是 Agent 的什么行为触发了越界。
- **测试绕过用例**：编写专门测试，用 `../`、`..\`、符号链接、多编码字符等进行验证，相当于对自己做的护栏进行“模糊测试”。

## 总结

给自动化脚本加目录白名单，成本极低（几十行代码），但对防止误操作和基础注入攻击非常有效。它不是万能的 —— 无法防住进程逃逸或内存攻击，但作为第一道工程护栏，能消除绝大多数与路径相关的意外。对于 OpenClaw 这类频繁扩展工具的 Agent 系统，这种防御性编程习惯值得推广。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/ea4c1ddd8dba88e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/bda369cea6f20528.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/c8f844631729c654.png)

