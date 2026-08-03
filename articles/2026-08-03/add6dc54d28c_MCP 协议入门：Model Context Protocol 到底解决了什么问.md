---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 31482
source: 综合讨论
publishedAt: 2026-08-03
---

## 先说背景：Agent 时代的信息孤岛

在 OpenClaw 这类 Agent 框架里，我们做的事情本质上是：**让模型能调用外部工具**。查天气、读写文件、调 API、操作数据库，每个工具都是一套独立接口。早期我们要为每个工具写 adapter——处理鉴权、拼参数、解析返回、错误重试。工具一多，这套胶水代码就成了负担。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），把它定位成“LLM 应用的 USB-C 接口”。到今天，这个协议已经被大量 Agent 框架和工具链采纳。它的核心思路是：**把“模型需要数据/动作”这件事统一成一条标准通道**，让 Agent 不需要关心每个工具背后的实现细节。

## MCP 到底解决了什么问题

如果只记一句话：**MCP 把“工具接入”从定制开发变成了标准化配置**。

具体拆解，它解决了三个痛点：

1. **接口割裂**：以前接一个工具要读一遍 API 文档，写针对性的调用层。现在工具方只需实现 MCP server，Agent 侧通过 MCP client 统一通信，协议帮你解决了传输、数据格式、错误处理。

2. **上下文失控**：大模型的 context 是稀缺资源。MCP 的 resource 和 tool schema 机制，让工具返回的数据可以被精确裁剪，只把“当前任务需要的那一部分”注入上下文，而不是一股脑全部交给模型。

3. **生态不互通**：以前每个 Agent 框架的工具生态是独立的。现在只要工具方支持 MCP，就意味着它能同时被 Claude Desktop、OpenClaw、Cursor 等支持 MCP 的客户端使用。写一次，到处跑。

## 快速跑通一个 MCP 服务

在 OpenClaw 里接 MCP server 通常就三步：写好 server、配置 client、验证调用。我们用一个最简单的时间查询服务做例子。

**第一步：写一个 MCP server**

用 Python 的 `mcp` 库，几十行就能写一个 server：

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import datetime

app = Server("time-server")

@app.list_tools()
async def list_tools():
    return [Tool(name="get_current_time", description="获取当前时间", inputSchema={"type": "object"})]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_current_time":
        return [TextContent(type="text", text=datetime.datetime.now().isoformat())]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**第二步：在 OpenClaw 中配置**

OpenClaw 支持通过 JSON 配置声明 MCP server（以 Claude Desktop 风格配置为例）：

```json
{
  "mcpServers": {
    "time-server": {
      "command": "python",
      "args": ["/path/to/time_server.py"]
    }
  }
}
```

**第三步：验证**

启动 OpenClaw，在对话里问一句“现在几点”，观察日志确认工具被调用，返回正确，上下文里只塞了时间字符串。

## 踩坑点（这些坑值得你注意）

**坑 1：stdio 超时的隐形陷阱**——如果你的 server 首次启动要加载模型或初始化连接（比如连数据库），要预留足够长的初始化时间。OpenClaw 的 client 默认超时可能只有 30 秒，你可以通过 `timeout` 参数调大。实际遇到过 server 启动要 40 秒，直接超时断开，日志里只有一条 `Connection closed`，排查起来很耗时间。

**坑 2：JSON-RPC 细节别自造轮子**——MCP 底层是 JSON-RPC 2.0，消息必须包含 `jsonrpc` 字段，id 要与请求对应。如果自己从零实现而非用官方 SDK，很容易在这些细节上出问题。建议：**除非有特殊需求，否则直接用官方 SDK**。

**坑 3：工具名冲突**——如果你的聚合场景里多个 server 暴露了同名工具（比如 `search`），OpenClaw 的处理策略是后加载的覆盖先加载的，容易产生“调用结果不符合预期”的诡异 bug。建议对每个 server 的工具名加前缀，在 server 侧就避免冲突。

**坑 4：同步 vs 异步**——`call_tool` 如果声明为 `async` 但内部跑了同步的阻塞调用（比如 `requests.get`），会阻塞整个事件循环，导致其他工具调用排队。建议耗时操作用 `asyncio.to_thread` 包一层，或者直接写同步函数交给框架调度。

## 可复用的建议

从 MCP 实践中可以沉淀出几条通用的工程经验：

- **用 MCP 做适配层**：如果你们内部有老系统的 HTTP/RPC 接口，不要让它直接暴露给 Agent。包一个薄薄的 MCP server，把内部接口包装成“语义化工具”——对模型友好，也方便内部做鉴权和审计。
- **schema 精简是王道**：工具的参数 schema 很大程度上决定了模型能不能正确使用它。原则是：**参数名要语义化**（用 `city_name` 而不是 `c1`），必填项越少越好，description 里附上合法的取值示例。
- **先把输入输出写死，再逐步松绑**：第一版工具可以把参数限定为枚举值，返回固定 JSON 结构。跑通全链路之后，再放开自由文本参数。出问题时二分定位：协议问题还是业务逻辑问题。
- **统一错误码约定**：MCP 的调用结果不管成功失败都走同一通道，建议在工具返回里定义 `code` 字段（如 `0` 成功，非零表示具体错误类型），让 Agent 能根据 code 做降级策略，而不是面对一坨异常堆栈。

## 总结

MCP 不是银弹，它也还在演进中——目前对于一些复杂场景（服务端主动推送、流式进度等）支持还不完善。但它至少把“Agent 接工具”这件事，从“每家自己做”变成了“业界有标准”。对 OpenClaw 用户来说，成本已经很低：SDK 齐全，配置简单。如果你手头有超过三个需要接入 Agent 的工具，MCP 值得投入去学习。

标准的意义不在于它有多么完美的设计，而在于它让生态里的参与者有了共同的对话基础。MCP 正在让这件事发生。

---

