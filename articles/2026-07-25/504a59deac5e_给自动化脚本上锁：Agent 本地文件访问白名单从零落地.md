---
title: 给自动化脚本上锁：Agent 本地文件访问白名单从零落地
feedId: 30358
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景：Agent 为什么需要一个“文件护栏”

不管是 OpenClaw 上挂接的 MCP 工具，还是跑在本地的自动化脚本，越来越多的 Agent 工作负载直接操作本地文件系统：读日志、写临时文件、搬运目录、清理缓存……在 Demo 环境里一切正常，但一旦放进真实工作目录，风险就出来了。

常见的坑：

- Agent 根据自然语言生成的文件路径不可靠，比如把 `~/Documents` 写成 `~/Docments`，然后创建出一堆“孤儿”目录；
- 一个本意是清理 `./tmp` 的脚本，因为传入错误参数，开始递归删除 `./` —— 再往后就是灾难；
- 插件或 MCP 工具本身没问题，但调用方传入的路径跨越了预期边界，比如读了一个包含敏感信息的配置文件；
- 本地脚本被其他进程或用户脚本注入，越权访问 `~/.ssh` 或系统配置。

这些问题都有一个核心共性：**文件操作没有边界约束**。给 Agent 加一个“本地目录白名单”机制，就是最直接、成本最低的护栏。

## 问题拆解：白名单必须解决什么

和传统 Web 应用不同，Agent 操作本地文件系统时，我们不能只靠“不写危险代码”的约束，因为路径来源往往是动态的（LLM 输出、上游工具结果、环境变量），必须在 IO 层做拦截。

一个可用的白名单方案至少要做到四点：

1. **严格解析路径**：处理相对路径、`~`、环境变量，统一为绝对路径；
2. **符号链接跟踪**：避免通过软链接逃逸到白名单外；
3. **读写权限分离**：某些目录只允许读，某些允许读写；
4. **可插拔集成**：不能要求开发者重写所有 `open()` 调用，需要最小的侵入性。

下面分享一个在 OpenClaw 环境下落地的方案，Python 实现 + 装饰器集成，同时给一个 MCP 工具的改造示例。

## 做法：从 Guard 类到 MCP 工具改造

### 1. 核心 Guard 类

核心思路：维护一组允许的目录前缀，对传入路径做规范化后检查。

```python
import os
from pathlib import Path

class FileAccessGuard:
    def __init__(self, read_whitelist=None, write_whitelist=None):
        self.read_dirs  = [Path(d).expanduser().resolve() for d in (read_whitelist or [])]
        self.write_dirs = [Path(d).expanduser().resolve() for d in (write_whitelist or [])]

    def _resolve(self, path: str) -> Path:
        return Path(path).expanduser().resolve()

    def _is_allowed(self, target: Path, allowed: list[Path]) -> bool:
        return any(target == d or d in target.parents for d in allowed)

    def check_read(self, path: str) -> Path:
        target = self._resolve(path)
        if not self._is_allowed(target, self.read_dirs + self.write_dirs):
            raise PermissionError(f"Read access denied: {target}")
        return target

    def check_write(self, path: str) -> Path:
        target = self._resolve(path)
        if not self._is_allowed(target, self.write_dirs):
            raise PermissionError(f"Write access denied: {target}")
        return target
```

要点：

- `expanduser() + resolve()` 一气呵成，解决 `~`、`..`、软链接攻击。
- `d in target.parents` 保证子目录也被放行（例如 `/data/project` 被允许，则 `/data/project/sub/file` 也可以访问）。
- 写权限列表自动包含读权限，避免重复配置。

### 2. 以装饰器形式嵌入现有脚本

对于现有自动化函数，可以直接装饰：

```python
guard = FileAccessGuard(
    read_whitelist=['~/logs', '/var/shared/config'],
    write_whitelist=['~/workspace/output', '/tmp/agent-tasks']
)

@with_guard(guard, read_args=[0], write_args=[1])
def process_file(src, dst):
    with open(src) as f_in:
        content = f_in.read()
    with open(dst, 'w') as f_out:
        f_out.write(content.upper())
```

`with_guard` 只是一个薄层的包装器，通过参数位置自动调用 `check_read` / `check_write`。这样老代码几乎零改动。

### 3. 集成到 OpenClaw / MCP 工具

如果你在用 OpenClaw 的插件或 MCP 工具（比如一个本地文件管理工具），建议**在工具接口层统一挂载 Guard**。

例如，原本的 MCP 工具 `write_file` 可能是这样：

```python
async def handle_write_file(path: str, content: str) -> dict:
    with open(path, 'w') as f:
        f.write(content)
    return {"status": "ok"}
```

改造后：

```python
guard = FileAccessGuard(write_whitelist=os.getenv('MCP_WRITE_WHITELIST', '').split(':'))

async def handle_write_file(path: str, content: str) -> dict:
    safe_path = guard.check_write(path)
    safe_path.parent.mkdir(parents=True, exist_ok=True)  # 确保目录存在
    safe_path.write_text(content)
    return {"status": "ok", "path": str(safe_path)}
```

白名单通过环境变量注入，部署时配置，不硬编码。同时可以在 Guard 中加日志记录，方便审计。

### 4. 边界测试

做一个快速的自测脚本：

```python
# 测试相对路径
print(guard.check_write('~/workspace/output/../../etc/passwd'))  # 应触发异常

# 测试软链接
# ln -s /etc/passwd /tmp/agent-tasks/link
print(guard.check_write('/tmp/agent-tasks/link'))  # 应触发异常
```

`resolve()` 会把 `..` 折叠，并跟踪软链接指向真实路径，逃逸行为会被准确拦截。

## 踩坑点

实际用下来，这几个点容易忽略：

1. **挂载点和 bind mount**  
   `resolve()` 会跨越文件系统边界，但如果你的白名单目录本身就是挂载点下的子目录，需要额外注意：`/mnt/data` 解析后可能变成 `/data`（如果做了 bind mount）。建议在初始化时打印出解析后的目录，提前验证。

2. **重复创建目录的副作用**  
   如上面例子中使用 `mkdir(parents=True)`，某些场景下这会意外创建不该存在的目录树。可以改为先检查父目录是否在白名单内，再决定是否创建。

3. **Windows 路径兼容**  
   如果 Agent 未来可能跨平台，`Path.resolve()` 在 Windows 上会处理盘符和 `\\`，但白名单列表需要统一使用 `\` 或 `/`？建议始终使用 `pathlib` 处理，不要手动拼字符串。

4. **性能开销**  
   对每个文件操作都做 `resolve()` 会有 I/O 开销（尤其是检查大目录树时）。对高频读写场景，可以对已解析路径做个 LRU 缓存，比如 `functools.lru_cache`。

## 可复用建议

- **配置化白名单**：通过环境变量或 YAML 配置文件注入，方便不同 Agent 实例使用不同权限集。
- **读写分离 + 只读默认**：只允许读的目录绝不给写权限，收敛风险。
- **统一入口**：如果项目里有多个工具，用一个全局 `FileAccessGuard` 实例，集中管理。
- **记录违规行为**：当拦截发生时，记录下完整路径、时间、调用来源（可用 `traceback` 或 OpenClaw 的日志上下文），方便回溯是否误配或攻击尝试。
- **与进程级沙箱配合**：白名单是应用层防护，还可以结合 `systemd` 的 `ReadOnlyPaths`、`ReadWritePaths` 或 Docker 的 volume 挂载来做第二道防线。

## 总结

Agent 文件访问白名単本质是在“能力”和“安全”之间加一层可编程的约束。它不复杂，但能防止绝大多数由路径不可控引发的误删、泄露问题。在 OpenClaw 这类平台上，把 Guard 直接嵌入 MCP 工具或插件层，改造代价很低，收益却很高。

项目初期就加上这层护栏，比事后加审计规则、恢复文件要舒服得多。希望这个实践能给你现在或未来的 Agent 项目一个安全的起点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/7d5527d6f1af7312.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/9c64b8ef1ed9c906.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/47d5e108acd1d6f4.png)

