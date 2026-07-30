---
title: Agent 文件访问护栏：用目录白名单实现安全隔离
feedId: 31004
source: 综合讨论
publishedAt: 2026-07-30
---

# Agent 文件访问护栏：用目录白名单实现安全隔离

当你给一个 Agent 接了文件读写能力，比如让它帮你整理下载目录、批量重命名图片、或者输出临时分析报告，紧接着冒出来的问题就是：它会不会在某个不经意的 prompt 下，删掉系统配置，或者把你还没备份的工程目录扫干净？

这不是危言耸听。在 OpenClaw 这类 Agent 框架中，挂载 MCP filesystem server 或者自己写的文件工具毫不费力，但默认的文件访问范围往往是整个文件系统。Agent 不具备安全意识，它把 `rm -rf /home/user/important` 和 `rm -rf /tmp/scratch` 当成同等操作。唯一能拉住它的，只有你在工具层预设的护栏。

**思路**：给文件访问工具加上一个**目录白名单**，Agent 只能读、写、删除白名单内的路径，其余一律拒绝。

---

## 1. 问题场景

假设你在 OpenClaw 中配置了以下工具：

- `read_file(path)` —— 读取文件内容
- `write_file(path, content)` —— 写入文件
- `delete_file(path)` —— 删除文件

这些函数直接透传路径给 Python 的 `open()` 或 `os.remove()`，没有任何限制。Agent 被要求“清理所有 .log 文件”，它可能会递归扫描整个根目录，然后……你猜。

即便 Agent 的 prompt 里写了“只操作 /workspace 目录”，模型依然可能幻觉、误解，或者在多轮上下文中被注入恶意路径。**工具层的强制校验比 prompt 约定可靠得多。**

---

## 2. 白名单检查器的实现

核心逻辑不复杂：拿到请求路径 → 解析为真实绝对路径 → 判断是否在白名单目录内。

我们用 Python 做一个最小实现，可以集成到任意工具函数中。

### 2.1 定义白名单

```python
import os
from pathlib import Path

# 白名单目录（绝对路径列表）
ALLOWED_DIRS = [
    "/app/workspace",
    "/app/tmp",
    "/app/data"
]
```

### 2.2 安全检查函数

```python
def is_safe_path(requested_path: str) -> bool:
    """返回 True 表示路径安全，可以操作。"""
    # 解析真实路径（消除符号链接、..、多余斜杠等）
    real_path = Path(requested_path).resolve()
    
    for allowed_dir in ALLOWED_DIRS:
        # 确保白名单目录本身也是真实路径
        allowed_real_dir = Path(allowed_dir).resolve()
        # 判断 real_path 是否在 allowed_real_dir 子树内
        # str 方式比较，注意尾部加分隔符防止 /app/workspace2 绕过
        if str(real_path).startswith(str(allowed_real_dir) + os.sep) or \
           str(real_path) == str(allowed_real_dir):
            return True
    return False
```

`resolve()` 这一步会彻底跟进符号链接、消除 `..`，把 `/app/workspace/../../../etc/shadow` 变成真实的 `/etc/shadow`，然后必然不匹配白名单。

### 2.3 包装工具函数

```python
def safe_read_file(path: str) -> str:
    if not is_safe_path(path):
        raise PermissionError(f"Access denied: {path}")
    with open(path, 'r') as f:
        return f.read()

def safe_write_file(path: str, content: str) -> None:
    if not is_safe_path(path):
        raise PermissionError(f"Access denied: {path}")
    # 对父目录必须存在或可创建进行额外检查（略）
    with open(path, 'w') as f:
        f.write(content)
```

在 OpenClaw 的工具注册中，将原来的 `read_file` 替换为 `safe_read_file`，就像给 Agent 安装了一个安全漏斗。

---

## 3. 集成到 MCP 文件服务器

如果你用的是官方的 `@modelcontextprotocol/server-filesystem`，它本身支持 `allowedDirectories` 参数，启动时指定即可：

```json
{
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-filesystem",
    "/app/workspace",
    "/app/data"
  ]
}
```

它内部也是做了类似的路径规范化检查。但如果你是在其它 MCP 服务器中自行实现文件操作，记住永远加上 `resolve()`。

---

## 4. 踩坑记录

- **符号链接绕过**：即使只允许 `/workspace`，Agent 如果能提前创建一个指向 `/etc` 的软链（如果允许写 `/workspace`），那么通过读取该软链就能读到 `/etc` 的内容。`resolve()` 可以化解这个攻击面，但前提是 Agent 没有在 `allowed_dirs` 外创建软链的权限。建议将白名单目录挂载成独立卷或 sandbox，杜绝原地创建捣乱软链。

- **Windows 的盘符和反斜杠**：如果 Agent 跑在 Windows 上，`resolve()` 会返回 `C:\workspace\data`，注意使用 `os.sep` 而不是硬编码 `/`，路径比较时要统一的大小写敏感性（推荐 `Path.match()` 但需要注意性能）。一个更稳健的跨平台做法是直接用 `real_path.relative_to(allowed_real_dir)` 判断是否抛出 `ValueError`。

- **白名单目录的父目录权限**：Agent 如果要创建新文件，父目录（白名单目录）需要有写权限。如果 Agent 运行在容器内，这通常不是问题。但注意不要在 `/tmp` 这类全局共享目录设置白名单，容易受其他进程影响。

- **未处理 race condition**：检查通过到实际打开文件之间，路径可能被替换。对于高安全场景，可以考虑在打开后再 `fstat` 并比较 `st_dev` 和 `st_ino`，确认文件确实在目标文件系统子树内。一般日常使用不必如此偏执，但留个心不亏。

---

## 5. 可复用建议

1. **白名单配置化**：通过环境变量 `ALLOWED_DIRS` 指定，工具启动时解析，避免硬编码。
2. **增加审计日志**：每次拒绝访问时记录时间、请求路径、解析后的真实路径，方便事后排查 Agent 的“越狱”企图。
3. **最小权限**：为 Agent 单独创建一个工作目录，例如 `/app/agent-data`，白名单里只放这一个目录。绝不包含用户主目录、`/etc`、`/bin`。
4. **禁止危险操作**：即便在白名单内，也可以考虑屏蔽 `os.system`、`subprocess` 等调用，防止 Agent 通过写入脚本再执行绕开限制。文件访问护栏是纵深防御的一层，不应单独依赖。

---

## 6. 总结

一个简短的 `is_safe_path` 函数就可以让 Agent 的文件操作从“全盘裸奔”变为“沙箱行走”。它没法替代完整的沙箱机制，但对大多数个人开发、自动化脚本、内部工具场景来说，已经能挡下 90% 的误操作和基础注入。工程上的安全感，往往就来自这些几行代码的小护栏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/16d7d05eaf9519fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/e8bba87ecf858280.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/3a2ce4b77ac5e08d.png)

