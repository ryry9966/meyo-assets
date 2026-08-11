---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 32671
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：当每个工具都要写一遍胶水代码

在用 OpenClaw 搭建 Agent 的过程中，你大概率遇到过这样的循环：模型需要查询天气 → 写一个 `get_weather` 函数 → 注册到工具列表 → 用 JSON Schema 描述参数 → 再写个 HTTP 适配器暴露出去。下次需要查数据库，又重复一遍，只不过函数名换成 `query_database`，授权改成环境变量。每个工具都是一个孤岛，工具描述、调用协议、错误返回格式各不相同。

更糟的是，当你把 Agent 从一个框架迁移到另一个，或者想让多个 Agent 共享同一批工具时，胶水代码几乎要全部重写。模型本身并不关心底层是 REST 还是 gRPC，它只需要一个结构化的上下文和工具定义。但在很长一段时间里，我们缺少一个通用层来解决“模型如何发现、描述并调用外部能力”的问题。

这就是 Model Context Protocol (MCP) 要切入的位置。

## 问题：碎片化的工具上下文与集成成本

Agent 系统中的核心矛盾可以归结为三点：

1. **工具元信息不统一**：有的用函数 docstring，有的用 OpenAPI 片段，有的手写 JSON Schema。模型解析这些格式的成本高，理解偏差容易导致调用失败。
2. **上下文注入方式混乱**：资源（如文件、数据库 schema）和工具常常混在一起，缺乏清晰边界。你很难定义“这些数据是只读参考”还是“可以执行操作”。
3. **传输层彼此割裂**：本地进程、HTTP、WebSocket、子进程……每一个通道都要求不同的序列化方式和生命周期管理。同一个工具在开发环境和生产环境的表现可能完全不一致。

MCP 并没有发明新的模型能力，而是把这三点抽象成一个客户端-服务器的标准协议：服务器端声明自己提供的资源 (Resources)、工具 (Tools)、提示 (Prompts)；客户端（也就是你的 Agent 进程）通过标准传输（目前主要是 stdio 和 SSE）连接到服务器，并用 JSON-RPC 2.0 通信。对模型来说，它看到的是一个稳定的、自描述的上下文集合，而不是散落各处的 API 碎片。

## 做法：搭建一个 MCP 工具并接入 Agent

下面用一个极简的天气查询工具来展示 MCP 的实际流程。我们使用 Python 的 `mcp` 官方 SDK。

**步骤 1：安装依赖**

```bash
pip install mcp
```

**步骤 2：编写 MCP 服务器**

```python
# weather_server.py
from mcp.server import Server, Tool
from mcp.types import TextContent
import httpx

server = Server("weather-service")

@server.tool()
async def get_weather(city: str) -> list[TextContent]:
    """Get current weather for a city. Returns temperature and condition."""
    # 模拟外部 API 调用，实际换成真实接口
    async with httpx.AsyncClient() as client:
        # 这里用占位数据演示
        data = {"city": city, "temperature": 22, "condition": "sunny"}
    return [TextContent(type="text", text=str(data))]

if __name__ == "__main__":
    server.run()
```

**步骤 3：在 OpenClaw 或通用 Agent 中配置 MCP 客户端**

OpenClaw 原生支持 MCP 客户端连接。在你的 OpenClaw 配置（通常是 `.openclaw.yaml` 或 UI 设置）中添加一个 MCP 服务器定义：

```yaml
mcp_servers:
  - name: weather
    command: python
    args: ["weather_server.py"]
```

或者对于远程服务器，指定 URL（SSE 模式）。客户端会在启动时自动拉起该进程，并通过 stdio 建立 JSON-RPC 通道。之后，Agent 就可以像调用内置工具一样使用 `get_weather`，无需额外注册。

**步骤 4：验证**

打开 OpenClaw 的调试面板或直接对话测试：“请查询北京的天气”，模型应当能识别工具、填入 city 参数并收到返回的内容。

## 踩坑点

在实际接入过程中，有几个地方容易卡住：

1. **工具描述的精确度**  
   工具函数的 docstring 会直接暴露给模型作为 tool description。如果描述模糊，比如只写“获取天气”，模型可能不知道该传入城市名还是坐标。务必在描述中明确参数含义和返回值结构。

2. **返回类型要符合 MCP 规范**  
   工具回调必须返回 `TextContent`、`ImageContent` 或 `EmbeddedResource` 的列表，不能直接返回 dict。如果你偷懒返回了 `{"temperature": 22}`，客户端解析时会报类型错误。

3. **异步与超时**  
   stdio 传输在协程调度下容易出现读取阻塞。如果工具内部使用了异步 HTTP 调用，确保 server.run() 使用的是正确的 asyncio 事件循环。有时会因为忘记 `asyncio.run()` 或事件循环冲突导致工具卡死，模型侧表现为长时间无响应。

4. **环境变量隔离**  
   通过 `command + args` 启动的服务器进程继承了 Agent 的环境变量。如果工具需要敏感的 API key，建议使用 `.env` 文件并在服务器端显式加载，而不是依赖全局变量，避免密钥意外泄露到模型上下文。

5. **工具参数 schema 自动推断的局限性**  
   MCP SDK 可以从 Python 函数签名自动生成 JSON Schema，但对于 Optional/Union 类型的处理有时不完美。建议用 Pydantic 模型作为参数类型，SDK 会自动推导出严格的 schema，减少模型幻觉。

## 可复用建议

- **把工具逻辑和 MCP 包装层分离**  
  核心业务函数（如真正的 API 调用）单独放在一个模块里，MCP 服务器只负责参数校验和结果序列化。这样你的工具可以同时用于 MCP 和其他非 MCP 场景（比如单元测试）。

- **用 MCP Inspector 调试**  
  MCP 官方提供了 `@modelcontextprotocol/inspector`，一个浏览器内调试工具。启动服务器后，你可以用 Inspector 可视化浏览所有工具和资源，手动触发调用并观察 JSON-RPC 消息。这比直接盲调 Agent 高效很多。

- **谨慎暴露 Resources**  
  Resources 用于提供只读上下文，比如数据库表结构或文档。不要将可修改的配置作为 Resource 暴露，以免模型误以为可以执行写操作。写操作统一走 Tools。

- **传输层选择建议**  
  本地单机 Agent 用 stdio（默认），零网络开销，权限可控。如果需要远程共享工具池，选用 SSE（Server-Sent Events）模式，并在反向代理层做好鉴权。避免将敏感工具直接暴露在公网。

- **版本化你的工具描述**  
  工具接口变化时，模型可能仍然缓存旧的 schema 描述。可以在工具名称中加入版本，如 `get_weather_v2`，或者利用 Prompt 机制通知模型更新说明。

## 总结

MCP 解决的不是“让模型变聪明”的问题，而是“让工具被模型正确地看见和使用”的问题。它把 Agent 开发中杂乱无章的胶水层抽象成一个统一的协议，降低了集成成本，也让不同 Agent 之间的工具复用成为可能。对于 OpenClaw 用户而言，一次编写 MCP 服务器，就能在多个 Agent 中复用，同时屏蔽传输与序列化细节。把精力从协议适配拉回到工具本身的可靠性上，这才是 MCP 带来的真正工程收益。

---

