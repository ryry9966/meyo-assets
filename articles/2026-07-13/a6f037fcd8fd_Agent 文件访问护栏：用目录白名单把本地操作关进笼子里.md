---
title: Agent 文件访问护栏：用目录白名单把本地操作关进笼子里
feedId: 28932
source: 综合讨论
publishedAt: 2026-07-13
---

# Agent 文件访问护栏：用目录白名单把本地操作关进笼子里

## 背景：当 Agent 有了读写本地文件的能力

无论是 OpenClaw 的插件生态、MCP 工具链，还是自己基于 LangChain/Agent 框架拼接的自动化脚本，我们越来越频繁地让 Agent 直接操作文件系统：读取配置、写入日志、生成报表、抽取数据。这种能力是把双刃剑——生产效率提上去了，但如果没有访问边界，Agent 的一条错误指令或幻觉输出就可能把 `.ssh`、`/etc` 甚至整个家目录搞得一团糟。

简单说：**你给 Agent 一把万能钥匙，就得亲手把不该开的门上锁。** 我最常采的落地方案就是“本地目录白名单”——Agent 只能读写你事先指定的目录及其子路径，其余位置一律拒绝。这篇文章不讲虚的，直接给一套可复用的 Python 实现与工程化踩坑记录。

## 问题拆解

我们需要一个安全中间层，在每次本地文件访问前做路径校验：

1. **白名单目录列表**：例如 `[“/data/project/uploads”, “./workspace”]`
2. **访问请求规范化**：将传入的相对路径、带 `..` 的路径、符号链接都解析为绝对路径的真实位置。
3. **越界判断**：确认规范化后的路径确实落在某个白名单目录之内，否则抛出异常或拒绝。
4. **与现有工具集成**：对 MCP 的本地工具、自动化脚本里的 `open()`、`shutil` 调用做一层透明包裹。

## 实现步骤

### 1. 核心校验函数

用 `pathlib` 解析路径，可以自动处理不同操作系统的分隔符和 `..`。但标准库的 `resolve()` 会跟随符号链接，这正是我们需要的——否则攻击者可通过软链接跳出白名单。

```python
import os
from pathlib import Path
from typing import List, Union

class FileAccessGuard:
    def __init__(self, allowed_dirs: List[Union[str, Path]]):
        # 预解析白名单，确保后续比对高效且可跨平台
        self.allowed_dirs = [Path(d).resolve() for d in allowed_dirs]

    def is_allowed(self, target: Union[str, Path]) -> bool:
        try:
            real_path = Path(target).resolve()
        except (OSError, RuntimeError) as e:
            # 路径无效或解析失败，直接拒绝
            return False

        # 判断 real_path 是否以任一白名单目录为前缀
        for allowed in self.allowed_dirs:
            try:
                real_path.relative_to(allowed)
                return True
            except ValueError:
                continue
        return False

    def guard_read(self, path: Union[str, Path]) -> Path:
        p = Path(path)
        if not self.is_allowed(p):
            raise PermissionError(f"读取路径被拒绝: {path}")
        return p

    def guard_write(self, path: Union[str, Path]) -> Path:
        p = Path(path)
        # 对于写操作，即使文件暂时不存在，其父目录也必须在白名单内
        if not self.is_allowed(p.parent):
            raise PermissionError(f"写入路径被拒绝: {path}")
        return p
```

注意：`is_allowed` 中用了 `relative_to` 捕获 `ValueError`，比字符串前缀判断更安全，能正确处理 `/data/project` 和 `/data/project_evil` 这种易混淆情形。

### 2. 与自动化脚本集成

场景：一个 Agent 工具函数需要读取指定文件内容，并限制只能访问项目工作区。

```python
guard = FileAccessGuard(allowed_dirs=["./workspace", "/tmp/agent_sandbox"])

def safe_read(filepath: str) -> str:
    safe_path = guard.guard_read(filepath)
    with open(safe_path, 'r', encoding='utf-8') as f:
        return f.read()
```

如果想要更透明，可以写一个 `SafeFileSystem` 类，镜像常见文件操作：

```python
class SafeFileSystem:
    def __init__(self, guard: FileAccessGuard):
        self.guard = guard

    def open(self, path, mode='r', **kwargs):
        if 'w' in mode or 'a' in mode or 'x' in mode:
            safe_path = self.guard.guard_write(path)
        else:
            safe_path = self.guard.guard_read(path)
        return open(safe_path, mode, **kwargs)
```

然后 Agent 侧只使用 `SafeFileSystem` 实例，杜绝裸 `open`。

### 3. 结合 MCP 工具插件

如果你给 OpenClaw 写 MCP 工具 server，在工具的入口处先初始化 guard，每个工具调用都经过一次校验。例如一个返回文件列表的工具：

```python
def list_directory(dir_path: str) -> list:
    safe_dir = guard.guard_read(dir_path)
    if not safe_dir.is_dir():
        raise NotADirectoryError(f"{dir_path} 不是目录")
    return [p.name for p in safe_dir.iterdir()]
```

## 踩坑记录

1. **Windows 盘符与大小写**  
   `pathlib.Path.resolve()` 会将盘符转为大写，比如 `c:\Users` 变成 `C:\Users`，白名单若用小写会导致比对失败。统一使用 `Path.resolve()` 处理白名单，并在初始化时标准化即可。

2. **竞赛条件（TOCTOU）**  
   检查通过后到实际 `open()` 之间，白名单目录的软链接可能被替换。这类攻击需要较高权限才能触发，常规 Agent 运行环境风险较低。如果实在敏感，可以使用 `os.open` + 文件描述符 + `os.fdopen` 并在打开后检查真实路径，但实现复杂度大增。一般场景维持现有检查即可。

3. **移动文件跳出白名单**  
   `shutil.move` 或 `os.rename` 可能将文件移出白名单范围。如果 Agent 工具需要移动文件，应同时校验源路径和目的路径都在白名单内，或明确禁止移动操作。

4. **相对路径陷阱**  
   传入 `“../../etc/passwd”` 会被 `resolve()` 正确还原，但前提是当前工作目录已经确定。如果你的 Agent 运行在动态 cwd 环境，最好在 guard 初始化时 `os.chdir` 固定到一个安全目录，或者在 `guard_read` 中先将相对路径转为基于固定根的绝对路径。

5. **符号链接循环**  
   `resolve()` 默认在 Python 3.6+ 会抛出 `RuntimeError`（Windows）或 `OSError`（Unix）如果遇到循环符号链接，我们在 `is_allowed` 中捕获并返回 `False`，安全合规。

## 可复用建议

- **配置化白名单**：将允许目录列表写入环境变量或配置文件，如 `AGENT_SANDBOX_PATH`，便于不同部署环境灵活调整。
- **审计日志**：每次越权拦截记录日志，包含时间、请求路径、调用栈，便于定位 Agent 异常行为。
- **与进程级限制结合**：在 Linux 下可用 `systemd` 的 `ReadWritePaths` 或 Docker 的 volume 挂载做第二道防线。

## 总结

Agent 的本地文件访问不该是“要么全禁、要么全开”，用几十行代码拉起的目录白名单护栏，成本极低，却能堵住绝大多数无心或恶意的路径穿越。工程实践里，这类安全中间层最好做成标准库，强制所有工具函数走同一入口，避免重复造轮子或绕过。如果你正给 OpenClaw 定制一套操作本地的 MCP 工具集，优先考虑把它固化成 `BaseTool` 的一部分，从第一天起就卡住边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/9f4dad33de0d073b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/86aee86c94ea9e10.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/b55a8e65cff16ca6.png)

