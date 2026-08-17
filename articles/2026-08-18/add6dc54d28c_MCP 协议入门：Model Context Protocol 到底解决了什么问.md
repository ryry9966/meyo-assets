---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 33647
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景：工具接入的“M×N”困境

在 Agent 和自动化实践里，我们都遇到过同一类问题：给模型接一个查数据库的工具、读文件的脚本，或者调内部 API 的动作，总是要针对不同客户端重复适配。

OpenAI function calling 一套格式，Claude tool use 一套格式，自建 workflow 又要另一套。工具作者每支持一个新客户端，就要多维护一个 SDK 或描述文件；客户端每接一个新工具，也要做一层转换。结果是 M 个工具、N 个客户端，容易变成 M×N 的适配成本。

MCP（Model Context Protocol）想解决的就是这件事：**用一套标准协议，把模型需要的工具、资源和提示统一描述和调用**。它不提升模型智能，也不替代 Agent 框架，只做“上下文接入层”的标准化。

## MCP 实际解决什么问题

MCP 把“外部上下文”抽象成三类原语：

- **Tools**：可执行动作，比如查天气、写文件、调 API。
- **Resources**：可读取的数据，比如文件内容、数据库记录、文档片段。
- **Prompts**：可复用的提示模板，帮助模型按固定格式执行任务。

对客户端来说，只需要实现 MCP 客户端；对工具方来说，只需要实现 MCP server。双方通过 JSON-RPC 通信，客户端可以自动发现工具列表、读取资源、调用工具。这比“每个平台单独写插件”要工程化得多。

## 一个最小可复现的 MCP Server

下面用 Python SDK 做一个只读工具：查询本地文件是否存在，并返回文件大小。先安装依赖：

```bash
pip install mcp
```

服务端代码保持简单：

```python
import os
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("file-checker")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="file_info",
            description="Check if a local file exists and return its size in bytes. Use this for read-only file inspection.",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Absolute or relative file path"}
                },
                "required": ["path"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name != "file_info":
        raise ValueError(f"Unknown tool: {name}")
    path = arguments["path"]
    if not os.path.exists(path):
        return [TextContent(type="text", text=f"File not found: {path}")]
    size = os.path.getsize(path)
    return [TextContent(type="text", text=f"File exists, size: {size} bytes")]

if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server
    async def main():
        async with stdio_server() as (read_stream, write_stream):
            await server.run(read_stream, write_stream, server.create_initialization_options())
    asyncio.run(main())
```

在 OpenClaw 或 Claude Desktop 等客户端里，通过 `mcpServers` 配置接入：

```json
{
  "mcpServers": {
    "file-checker": {
      "command": "python",
      "args": ["/path/to/file_checker_server.py"],
      "env": {}
    }
  }
}
```

配置后，客户端启动时会自动执行 `initialize`、`tools/list`，模型需要时再发起 `tools/call`。

## 踩坑点

1. **stdio 与 SSE/HTTP 别混用**  
   本地命令行工具用 stdio 足够，但远程多客户端必须用 SSE 或 HTTP transport，并补鉴权、CORS、心跳。不要把 stdio server 直接暴露成网络服务。

2. **stdout 只能走协议消息**  
   stdio 模式下，服务端日志必须写 stderr。如果 print 到 stdout，会污染 JSON-RPC 消息流，客户端解析直接失败。

3. **JSON-RPC 错误要结构化**  
   工具内部出错时，不能只抛异常或打印日志。要返回带 `code` 和 `message` 的标准错误响应，否则客户端可能卡在超时等待。

4. **input schema 不严格，模型会乱填**  
   参数描述里写清楚类型、格式、边界。比如路径用“绝对路径”还是“相对路径”，是否支持 `~`，都要说明。模糊描述会导致模型调用参数漂移。

5. **环境变量不会自动继承 GUI 客户端**  
   如果 MCP server 依赖 `API_KEY`，必须在 `mcpServers` 配置里显式传 `env`。系统环境变量对图形客户端不一定可见。

6. **大结果撑爆上下文**  
   Resources 或工具返回过长文本，会占用大量 token，甚至触发上下文截断。服务端要限制返回长度、做分页或截断摘要。

## 可复用建议

- **先做只读工具，稳定后再开写操作**。写文件、删数据、执行命令都应有白名单和确认机制。
- **用 MCP Inspector 调试**，不要直接在正式客户端里反复试错。Inspector 可以单独发 `tools/list`、`tools/call`，排障效率高很多。
- **工具描述也是 prompt 工程的一部分**。写清楚“何时用、不用会怎样、返回什么格式”，模型调用准确率会明显提升。
- **给工具加命名空间前缀**。多个 server 都提供 `search` 时，改成 `local_search`、`db_search`，避免客户端选择混乱。
- **限制文件访问根目录**。如果必须读本地文件，只允许访问指定目录，并对路径做规范化校验，防止 `../` 越权。
- **固定 SDK 版本**。MCP 规范仍在演进，升级前先看 changelog，避免协议不兼容。

## 总结

MCP 解决的是工具、资源和提示的接入标准化问题，让 Agent、OpenClaw 和插件作者减少重复适配。它不负责推理、不负责编排，也不保证模型一定把工具用对。

工程上的收益是：一次编写 MCP server，可以在多个客户端复用；工具生态从“平台专用”逐步走向“协议通用”。但要落地好，仍然需要处理好传输方式、错误结构、权限边界和性能限制。

务实路线可以这样走：**先本机、后远程；先只读、后写操作；先工具、后资源；先跑通最小闭环，再扩展能力。**

---

