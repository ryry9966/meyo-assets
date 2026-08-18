---
title: MCP 协议入门：它替 Agent 工具接入解决了什么实际问题
feedId: 33796
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景：以前接工具为什么难受

在做 Agent、OpenClaw 插件或自动化时，最耗精力的往往不是模型本身，而是把外部能力接进来。比如接文件系统、浏览器、数据库、搜索 API，每次都要写一套适配：参数 schema、鉴权、错误重试、结果解析、日志。换一个 Agent 框架，很多代码又得重写。

MCP（Model Context Protocol）要解决的就是这个集成层的问题。它把“模型如何使用工具、读取数据、调用提示”标准化，让外部能力以 MCP Server 的形式暴露，Agent 侧只需要一个 MCP Client。

## MCP 到底解决什么

它解决的并不是“模型变聪明”，而是把 N 个工具 × M 个平台 的适配问题，降到 N 个 Server + 1 个 Client。核心约束的是通信层和描述方式：

- 基于 JSON-RPC 2.0，支持 stdio / HTTP+SSE；
- 用 `tools/list`、`tools/call`、`resources/read` 等方法交互；
- 每个工具必须有名称、描述、JSON Schema 参数定义。

对 OpenClaw 用户来说，这意味着：你维护一个 MCP Server，可以在不同客户端里复用，不用再为每个框架写一次性插件。

## 最小实践：在 OpenClaw 侧接一个 MCP Server

这里用 Python 的 `mcp` SDK 做一个最小闭环。

### 1. 写一个 Server

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Return the sum of two integers. Use this when the user asks to add two numbers."""
    return a + b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

保存为 `demo_server.py`。

### 2. 客户端配置

在 OpenClaw 的 MCP 配置里加上：

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["demo_server.py"],
      "cwd": "/path/to/your/project"
    }
  }
}
```

启动后，Agent 侧应该能看到 `demo` 这个 Server，以及它提供的 `add` 工具。描述写清楚，模型才知道什么时候调用。

### 3. 验证

在接入 Agent 前，建议先用 MCP Inspector 或 `mcp dev` 单独测 Server，确认 `tools/list` 能返回工具、`tools/call` 能正常执行。先跑通 stdio，再考虑 HTTP/SSE。

## 踩坑点

- **stdio 输出污染**：Server 里如果直接用 `print()` 输出日志，会破坏 JSON-RPC 帧。日志必须写到 stderr，否则客户端解析失败。
- **参数 Schema 不严格**：`int`、`float`、`Optional` 要定义清楚。模型调用时如果 schema 模糊，很容易传错类型。
- **描述太短或太泛**：工具描述不是给开发者看的，是给模型看的。要写清楚“什么时候用、参数含义、返回什么”。只写“查询数据”基本等于没用。
- **路径和启动命令**：Windows 下 `npx` 有时需要 `cmd /c` 包一层；Python 要确认用的是虚拟环境里的解释器，否则依赖找不到。
- **返回必须可序列化**：不能直接返回 `datetime`、自定义对象等不可 JSON 序列化的内容。
- **长任务卡住 Agent**：如果工具执行时间过长，Agent 会一直等。Server 内部要做超时、分页或异步处理。
- **版本兼容**：MCP 协议和 SDK 演进较快，尽量固定客户端与 Server 的 SDK 版本，避免 `tools/call` 格式变化导致不兼容。

## 可复用建议

- 先做最小闭环：一个 Server、一个工具、stdio、本地，跑通后再扩展。
- 把工具描述当 prompt 写，包含使用场景和限制条件。
- 每个 MCP Server 保持单一职责，比如 `filesystem`、`browser`、`database` 分开维护。
- 用 MCP Inspector 做回归测试，尤其是参数 Schema 和返回结构变化时。
- 生产环境如果用 SSE/HTTP 暴露，要加鉴权、限流和超时控制；stdio 适合本机或容器内使用。

## 总结

MCP 的价值在于把 Agent 与外部系统的集成标准化，降低重复适配成本。它不是银弹：工具设计质量、权限控制、稳定性仍然要自己负责。但对 OpenClaw 这类实践用户来说，与其继续写一次性适配器，不如把已有能力封装成 MCP Server，至少后续换客户端、换 Agent 框架时，不用再重写一遍。

---

