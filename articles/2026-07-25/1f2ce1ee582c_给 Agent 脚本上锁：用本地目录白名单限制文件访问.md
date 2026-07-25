---
title: 给 Agent 脚本上锁：用本地目录白名单限制文件访问
feedId: 30389
source: 综合讨论
publishedAt: 2026-07-25
---

## 问题场景

在 OpenClaw 这类 Agent 框架里，模型调用工具（MCP 服务器 / 本地插件）执行文件读写已经是很常见的模式。一个简单的“帮我把下载目录里所有 PDF 重命名为日期格式”的 prompt，背后就可能触发 `list_files` → `move_file` 链式调用。如果没有任何护栏，模型幻觉、prompt 注入或者工具描述偏差，都可能导致 Agent 访问到不该碰的路径——比如 `~/.ssh`、`/etc/passwd` 或者项目源码之外的数据目录。

本地文件系统不像云函数那样天然隔离，一旦给了 Agent 删除或覆盖权限，误操作代价很高。与此同时，我们又希望 Agent 具备真正的自动化能力，不能因为害怕就直接关掉文件工具。所以需要在“允许操作”和“安全可控”之间找到工程化的平衡点。

## 设计思路：目录白名单 + 路径规范化

最直接有效的做法是：**在工具实现层加一层文件访问白名单，只允许 Agent 操作预先指定的目录子树**。这比单纯字符串黑名单（比如禁止包含 `..` 或 `/etc`）靠谱得多，因为绕过黑名单的方法太多：符号链接、硬链接、大小写绕过、Windows 短文件名等等。白名单则从“允许哪些根目录”这个前提开始约束，凡是解析后的绝对路径不在白名单子树内，一律拒绝。

这个白名单可以放在三个位置：

1. **MCP 服务器内部**：在工具函数入口做路径校验。
2. **Agent 运行时的工具封装层**：比如在 OpenClaw 的 `tool` 装饰器里统一拦截。
3. **操作系统级隔离**：用 systemd 服务文件、AppArmor 或 Docker 卷挂载实现。

实际项目中，我建议至少把 **1+3 组合**，即应用层校验保证逻辑正确，OS 层隔离作为纵深防御。下面以 Python MCP 服务器为例，给出一套可直接复用的白名单校验模式。

## 实现步骤

### 1. 定义白名单根目录

在配置文件中声明允许访问的目录列表，不要硬编码。例如：

```yaml
# agent_config.yaml
file_whitelist:
  - /home/user/agent-workspace
  - /tmp/agent-sandbox
```

运行时读取这些路径，全部通过 `os.path.realpath` 解析成规范化的绝对路径，存为一个元组。

```python
import os
from pathlib import Path

def load_whitelist(paths: list[str]) -> list[Path]:
    resolved = []
    for p in paths:
        real = Path(os.path.realpath(p))
        if not real.exists():
            raise FileNotFoundError(f"Whitelist path not found: {real}")
        resolved.append(real)
    return resolved
```

### 2. 路径校验函数

所有 MCP 工具在访问文件前，必须调用 `validate_path`，并且只接受解析后的真实路径做比对。

```python
def validate_path(
    user_path: str,
    whitelist: list[Path],
    must_exist: bool = False
) -> Path:
    target = Path(user_path)
    if must_exist:
        if not target.exists():
            raise FileNotFoundError(f"Path does not exist: {target}")
    real_target = Path(os.path.realpath(target))
    # 检查是否在任一白名单目录内
    for allowed in whitelist:
        try:
            real_target.relative_to(allowed)
            return real_target
        except ValueError:
            continue
    raise PermissionError(
        f"Access to {real_target} is not allowed by whitelist"
    )
```

关键点：**必须使用 `os.path.realpath` 解析符号链接和 `..`**。否则攻击者可以通过 `～/.ssh` 的符号链接指向 `/etc` 绕过白名单。`relative_to` 在 Python 3.9+ 会严格检查路径是否为子路径，不会出现 `/tmp/../etc` 绕过的情况。

### 3. 在 MCP 工具中集成

以读取文件工具为例：

```python
@server.call_tool()
async def read_file(path: str) -> str:
    safe_path = validate_path(path, whitelist, must_exist=True)
    return safe_path.read_text(encoding="utf-8")
```

写入文件同理，在写操作前加上相同校验。注意写操作可能创建新文件，此时 `must_exist=False`，但仍需要确保父目录在白名单内，并检查不通过符号链接跳出。`validate_path` 已经覆盖了新文件的情况，因为它解析的是最终真实路径，即使文件尚不存在，也能检查解析后的路径是否处于白名单中。

### 4. 操作系统级兜底

为 Agent 进程配置专门的系统用户，禁止访问非授权目录。Linux 下可以结合 systemd 服务设置：

```ini
[Service]
User=agent
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/user/agent-workspace /tmp/agent-sandbox
```

或使用 Docker 启动，只挂载白名单目录：

```bash
docker run -v /home/user/agent-workspace:/workspace:rw \
           -v /tmp/agent-sandbox:/tmp/sandbox:rw \
           my-agent-image
```

这样即使应用层校验被突破，进程也没有权限读取其他系统路径。

## 踩坑记录

**符号链接陷阱**  
最常犯的错误是用 `os.path.abspath` 替代 `realpath`。`abspath` 只拼接当前工作目录，不解析符号链接。假设白名单是 `/workspace`，用户传入 `/workspace/link_to_etc`，符号链接指向 `/etc`，`abspath` 会得到 `/workspace/link_to_etc`，通过白名单检查，而实际操作会落到 `/etc` 下。解决方案就是强制 `realpath`，并确保白名单路径本身也经过 `realpath` 处理。

**相对路径与工作目录**  
如果 Agent 在子进程中执行脚本，当前工作目录可能不是白名单根。传入 `../../secret` 这类相对路径时，`validate_path` 如果先 `resolve()` 再检查，就能正确还原真实路径。所以永远不要用 `os.path.join(workspace, user_input)` 的简单拼接，必须走规范化流程。

**跨平台兼容**  
Windows 上 `realpath` 的行为与 Linux 略有不同，但 `pathlib.resolve()` 基本等价。如果服务需要跨平台，统一用 `Path.resolve()`，它会解析符号链接和 `..`。另外 Windows 驱动器字母也是路径的一部分，白名单配置时需要注意。

## 可复用建议

1. **将 `validate_path` 抽象成独立模块**，任何 MCP 工具（文件读写、执行命令的 `cwd` 参数）都可以调用。
2. **把白名单配置放在 Agent 全局上下文**，让所有需要文件访问的工具共享同一份校验逻辑，不要每个工具自己实现一遍。
3. **在 CI 中加入路径安全测试**：故意传入符号链接、`..`、绝对路径指向系统目录，确认工具返回 `PermissionError`。
4. **增加日志记录**：凡是校验失败的请求，记录用户传入的原始路径、解析后的路径和时间戳，便于事后排查是误报还是攻击尝试。
5. **最小权限原则**：不要给 Agent 目录写权限，除非业务真的需要。许多自动化场景只需要读取文件，就可以把白名单路径挂载为只读。

## 总结

给 Agent 加上本地目录白名单，不是过度防御，而是让自动化脚本从“玩具”走向“生产”的必要条件。实现上并不复杂，核心就是 **路径规范化 + 白名单前缀匹配**，再配合操作系统层的访问控制，可以拦住绝大多数意外的文件暴露风险。在 OpenClaw / MCP 的工程实践中，这套逻辑建议直接内置到工具基类里，让后续所有的文件操作都天然受到约束，避免逐工具修补的混乱。

安全护栏越是自然、无感，Agent 的自动化能力反而越敢放手使用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/98c3c5d9c0b38e86.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/792eafd3a8e53f2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/b73b6bb99a5075e1.png)

