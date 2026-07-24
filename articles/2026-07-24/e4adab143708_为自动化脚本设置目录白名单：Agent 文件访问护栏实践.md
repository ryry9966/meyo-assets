---
title: 为自动化脚本设置目录白名单：Agent 文件访问护栏实践
feedId: 30266
source: 综合讨论
publishedAt: 2026-07-24
---

# 为自动化脚本设置目录白名单：Agent 文件访问护栏实践

## 背景与动机

在以 OpenClaw、MCP 或自定义 Agent 搭建自动化流程时，常常需要赋予脚本读写本地文件的能力——生成日志、加载配置、导出数据文件。一旦 Agent 能够通过函数调用或插件访问文件系统，安全边界就会被打开：一个错误的提示、一段未经验证的指令，都可能让脚本逾越预期范围，读取敏感信息或写入系统目录。

语言模型驱动的自动化场景里，我们没有“用户意图”以外的防线。文件访问权限本质上由**运行进程的用户**控制，而不是由任务上下文控制。因此，如果希望 Agent 只在某个项目目录及其子目录下活动，就需要在代码层实现显式的访问护栏。

本文将介绍一种轻量、工程化的本地目录白名单方案，适合集成到任何基于 Python 的 Agent 或插件中。不依赖重量级沙箱，只通过路径规整、前缀匹配与少数预防符号链接绕过的检查，确保所有文件操作都被限制在许可范围内。

## 问题定义

目标：给定一个或多个“合法根目录”（白名单），任何绝对/相对路径的读写请求，必须在最终解析后的绝对路径上，满足“属于白名单目录子树”的约束。这需要处理如下问题：

- 相对路径的规范化（依赖当前工作目录）
- 符号链接可能把路径引向白名单外
- 路径遍历攻击（`../`）
- 跨平台路径分隔符差异

如果只做简单的字符串前缀检查，很容易因为符号链接或相对路径绕过。因此，正确的做法是 **“解析真实绝对路径后再做目录包含判断”**。

## 实现步骤

### 1. 定义白名单配置

白名单最好来自配置文件或环境变量，避免硬编码。例如，在 `config.yaml` 中：

```yaml
file_guard:
  allowed_roots:
    - /home/user/project/data
    - /tmp/agent_workspace
```

在代码中加载为列表。

### 2. 路径安全解析函数

写一个函数，将任意输入路径解析为不包含符号链接的绝对路径，并检查是否在白名单子树内。核心 Python 实现如下：

```python
import os
from pathlib import Path

def resolve_safe_path(raw_path: str, allowed_roots: list[str]) -> Path:
    # 展开用户目录 ~
    expanded = os.path.expanduser(raw_path)
    # 将相对路径解析为绝对路径，但不解开符号链接
    abs_path = os.path.abspath(expanded)
    # 使用 realpath 解析所有符号链接，返回规范绝对路径
    real = os.path.realpath(abs_path)
    real_path = Path(real)

    # 检查是否在任一白名单根目录下
    for root in allowed_roots:
        root_real = Path(os.path.realpath(root))
        try:
            # Python 3.9+ 支持 is_relative_to
            if real_path.is_relative_to(root_real):
                return real_path
        except AttributeError:
            # 低版本用 commonpath 比较
            if os.path.commonpath([str(root_real), str(real_path)]) == str(root_real):
                return real_path

    raise PermissionError(f"Access denied: {real_path} is outside allowed roots")
```

**关键点：**
- `os.path.realpath` 会消除所有符号链接，得到最终目标的真实路径，避免链接绕过。
- `is_relative_to` 是 Python 3.9 引入的路径工具，安全判断目录归属关系。低版本可用 `os.path.commonpath` 替代，但需额外检查共同路径完全等于白名单根，而不能只是前缀，防止 `/tmp/workspace` 误包含 `/tmp/workspace_evil`。

### 3. 封装文件操作

在 Agent 的工具函数或 MCP 服务接口中，所有涉及路径的参数都先过护栏。例如，对于文件读取工具：

```python
def safe_read(file_path: str, allowed_roots: list[str]) -> str:
    safe_path = resolve_safe_path(file_path, allowed_roots)
    with open(safe_path, 'r') as f:
        return f.read()
```

如果使用装饰器可以进一步消除样板：

```python
def guarded(allowed_roots):
    def decorator(func):
        def wrapper(file_path, *args, **kwargs):
            safe_path = resolve_safe_path(file_path, allowed_roots)
            return func(safe_path, *args, **kwargs)
        return wrapper
    return decorator
```

然后对工具函数加 `@guarded(allowed_roots)`。

### 4. 与 MCP 文件系统工具集成

若你使用的是 MCP filesystem 服务器（例如 `@modelcontextprotocol/server-filesystem`），它本身已提供类似的 `allowedDirectories` 配置项，只需启动时指定即可：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/project/data",
        "/tmp/agent_workspace"
      ]
    }
  }
}
```

该官方实现内部也已处理符号链接和路径遍历问题，你可以直接复用。但如果要定制更多逻辑，自己实现一套轻量护栏仍是首选。

## 实践过程中的坑点

1. **符号链接逃脱**  
   初版实现只用了 `os.path.abspath`，没有用 `realpath`，导致白名单目录内的软链接指向外部时被放行。务必使用 `realpath`，并且对白名单根目录本身也做 `realpath` 归一化，否则两边不一致。

2. **`commonpath` 陷阱**  
   若不使用 `is_relative_to`，而用 `commonpath`，必须比较返回值与白名单根目录完全相同，而不是简单的 “以白名单开头”。否则 `/tmp/a` 白名单会错误地允许 `/tmp/ab/../etc/passwd`（如果用户输入被构造）。正确做法：
   ```python
   if os.path.commonpath([root, str(real_path)]) == root:
       ...
   ```

3. **Windows 盘符大小写**  
   若跨平台，注意 `os.path.realpath` 返回的盘符大写或小写可能不一致，需在比较前统一转为小写，并确保白名单根盘符标准化。

4. **竞态条件**  
   如果流程是：先检查路径合法 -> 后打开文件，中间存在时间窗口，路径目标可能被替换。对于一般自动化场景，风险较低，但如果处理特权操作，最好在打开后再次通过文件描述符确认路径（如使用 `os.fstat` 与预期的目录比较），或使用 `open` 的 `dir_fd` 参数在目录 fd 下操作以确保不离开。普通护栏工程中可暂不处理，但需知晓。

5. **临时文件与日志**  
   如果你允许 Agent 使用 `tempfile` 或日志库，其默认路径可能在 `/tmp` 等系统目录。需要在白名单中显式加入这些路径，或通过环境变量将临时目录约束到项目目录内。

## 可复用建议

- **统一入口函数**：不要在多处直接调用 `open`，而是通过 `safe_open`、`safe_write` 等封装，强制所有文件操作经过护栏。
- **配置分层**：允许白名单针对不同 Agent 或插件实例分别指定，以避免权限交叉污染。
- **记录审计日志**：在 `resolve_safe_path` 中记录每次解析的原始路径、真实路径及是否允许。这对调试和回溯非常有用。
- **添加干运行模式**：在开发阶段，可以让护栏只记录拒绝警告而不真正抛出异常，便于发现遗漏的白名单需求。
- **测试边界**：编写单元测试覆盖符号链接、相对路径、`~/`展开、路径遍历，以及 Windows/Linux 差异。

## 总结

给自动化脚本加上目录白名单，本质是用“解析真实路径 + 归属检查”替代粗糙的前缀黑名单。实现成本很低，但可以极大缩小 Agent 文件操作的攻击面。在 MCP 生态中，可以直接利用官方 filesystem 服务器的成熟实现；定制脚本中，用不到 50 行 Python 代码就能建成可靠护栏。

工程化实践的核心是：永远不信任传入的路径字符串，始终求实数并验证，同时保持配置灵活、日志完备。在防止 Agent“越界”的同时，不影响其完成常规数据读写任务。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/0b9223dfae97ad40.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/f2da90647cdc1a60.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/1d53dc104ee26238.png)

