---
title: Agent 文件操作护栏实践：为自动化脚本设置目录白名单
feedId: 30945
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 OpenClaw 这类 Agent 框架中，让 LLM 通过执行本地脚本或调用系统命令来完成自动化任务是常见需求。无论是通过内置的 Shell 工具，还是通过 MCP（Model Context Protocol）暴露的文件服务，Agent 一旦获得文件系统访问能力，就可能面临“操作越界”的风险：一条错误的 `rm -rf` 或一次对 `/etc/passwd` 的读取尝试，都会造成难以预期的后果。

因此，我们需要一种轻量且可控的护栏机制，让 Agent 只能在指定的目录范围内读写文件，既保留自动化能力，又不暴露关键目录。本文将介绍一种基于目录白名单的文件访问护栏方案，并用 Python 实现的 MCP 文件服务作为示例，可直接在 OpenClaw 中集成使用。

## 问题分析

典型的 Agent 文件操作链路是：用户指令 → Agent 推理 → 调用工具（如 `read_file`、`write_file`）→ 本地文件系统。如果在工具层不做限制，以下操作都可能成为安全隐患：

- 读取敏感配置文件（如 `~/.ssh/id_rsa`）
- 覆盖关键系统文件（如 `/etc/hosts`）
- 使用 `..` 或符号链接逃逸到白名单外目录
- 通过绝对路径直接访问任意位置

给工具加上“目录白名单”是最直接有效的约束方式：只有解析后的真实路径落在白名单内时，才允许读写。这样即使 Agent 产生幻觉或受到注入攻击，其破坏范围也会被限定在安全区域内。

## 方案设计

核心思路：在文件操作的 MCP 工具中，对传入的路径参数进行规范化解析，并检查其是否以预设的白名单目录为前缀。具体要点如下：

- 白名单目录在服务启动时指定，可配置多个目录，例如 `["./workspace", "/tmp/agent-sandbox"]`。
- 路径规范使用 `os.path.realpath` 或 `pathlib.Path.resolve()`，消除所有 `..` 和符号链接。
- 检查规范路径是否在任一白名单目录内（以目录分隔符结尾的前缀匹配）。
- 对于读、写、列表目录等操作，统一应用此检查。
- 若校验失败，返回明确错误信息，记录日志用于审计。

为什么不用简单的字符串前缀匹配？因为 `"./workspace"` 可能被 `"./workspace/../etc/hosts"` 绕过，而符号链接也会导致路径指向意外位置。必须解析真实路径后再做前缀判断。

## 实现步骤（基于 MCP + Python）

### 1. 项目结构

```
fileguard-mcp/
├── server.py          # MCP 服务端，含白名单校验逻辑
├── requirements.txt   # mcp, pydantic 等依赖
└── README.md
```

`requirements.txt` 内容：
```
mcp[cli]
pydantic
```

### 2. 编写 MCP 服务端

使用 `mcp` 库的 `FastMCP` 快速搭建工具，示例代码如下（已精简，仅保留核心逻辑）：

```python
import os
from pathlib import Path
from mcp.server import FastMCP

# 白名单目录列表，可通过环境变量或配置文件传入
ALLOWED_DIRS = [Path(p).resolve() for p in os.environ.get("FILEGUARD_DIRS", "./workspace").split(":")]

app = FastMCP("fileguard")

def is_path_allowed(path_str: str) -> Path:
    """检查路径是否在白名单内，返回解析后的路径，否则抛出 ValueError"""
    target = Path(path_str).resolve()
    for allowed in ALLOWED_DIRS:
        # 确保 allowed 目录以分隔符结尾，防止 /var/app 匹配 /var/app2
        if str(target).startswith(str(allowed) + os.sep) or target == allowed:
            return target
    raise ValueError(f"Access denied: {target} is outside allowed directories.")

@app.tool()
def read_file(path: str) -> str:
    """读取文件内容，仅限白名单目录"""
    safe_path = is_path_allowed(path)
    return safe_path.read_text()

@app.tool()
def write_file(path: str, content: str) -> str:
    """写入文件，仅限白名单目录"""
    safe_path = is_path_allowed(path)
    safe_path.write_text(content)
    return "Write successful"

if __name__ == "__main__":
    app.run()
```

以上代码中，`is_path_allowed` 是核心护栏函数。请注意以下几点工程化细节：

- 使用 `resolve()` 先获取绝对路径，防止相对路径和符号链接绕过。
- 在比较前缀时，为每个 `allowed` 目录手动追加 `os.sep`，避免 `/app` 匹配到 `/app-private` 这种误判（除非精确相等）。
- 异常会被 MCP 框架捕获并返回给 Agent，便于调试。

### 3. 配置 OpenClaw 使用该 MCP 服务

在 OpenClaw 的 MCP 配置文件中注册该服务：

```json
{
  "mcpServers": {
    "fileguard": {
      "command": "python",
      "args": ["path/to/server.py"],
      "env": {
        "FILEGUARD_DIRS": "./project_data:/tmp/agent-scratch"
      }
    }
  }
}
```

这样，Agent 在调用 `read_file` 或 `write_file` 时，只能访问 `./project_data` 和 `/tmp/agent-scratch` 两个目录下的文件。

## 踩坑点

1. **符号链接陷阱**  
   即使设置了白名单，用户或 Agent 可能故意在工作目录内创建指向 `/etc` 的符号链接。使用 `resolve()` 可以解析链接目标，从而拒绝访问。务必使用真实路径而非相对路径。

2. **路径分隔符问题**  
   前缀匹配时若不加分隔符，`/data/projectA` 可能匹配到 `/data/projectA_secret`。正确做法是始终在比较时给白名单目录追加 `os.sep`，除非两路径完全相同。同时注意 Windows 与 POSIX 的差异。

3. **目录存在性**  
   `resolve()` 在路径不存在时会返回假设的绝对路径（保留不存在的部分），虽可解决“提前检查”场景，但若 Agent 尝试在未创建的目录写入文件，依然需要白名单目录真实存在才能访问。建议在启动时确保白名单目录已创建。

4. **并发问题**  
   多个 Agent 实例共享同一个 MCP 服务时，文件操作可能发生冲突。加上简单的文件锁或通过任务队列可避免，但超出本文范围。至少应保证白名单逻辑自身无状态。

5. **日志与告警**  
   否决访问时，除了返回错误，还应记录请求路径、Agent ID 和时间戳，方便事后排查和感知异常行为。

## 可复用建议

- **白名单配置外部化**：使用环境变量或配置文件，避免硬编码，便于部署时调整。
- **封装成通用校验器**：将 `is_path_allowed` 抽象为独立模块，方便在其他工具（如 Shell 命令）中复用，例如在执行 `os.system` 前检查命令行参数中的路径。
- **与现有沙箱结合**：对于高风险场景，可搭配 Docker/容器级别的文件系统隔离，目录白名单作为应用层额外的一层防御深度。
- **测试路径边界**：写单元测试覆盖`..`穿越、符号链接、相对路径、不存在的路径等情况，确保校验函数健壮。

## 总结

给 Agent 的文件操作加上目录白名单，是一种低成本但有效的权限控制策略。它不能替代完整的沙箱，但足以避免大部分因 Agent 误操作导致的数据泄露或系统损坏。在 OpenClaw 的生态中，利用 MCP 的自定义工具能力，我们可以快速实现一个可配置的文件访问护栏，让自动化脚本在受控的环境中发挥价值。始终牢记“最小权限”原则，只给予 Agent 完成当前任务所必需的目录访问权。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/d3f721d203897323.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/0da402d4cceecd87.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/4e356c01229b144a.png)

