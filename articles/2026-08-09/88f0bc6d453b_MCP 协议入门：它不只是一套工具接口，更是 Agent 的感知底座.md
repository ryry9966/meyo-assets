---
title: MCP 协议入门：它不只是一套工具接口，更是 Agent 的感知底座
feedId: 32182
source: 综合讨论
publishedAt: 2026-08-09
---

## 一、背景：Agent 的工具集成，为什么总在“造轮子”

如果你维护过包含工具调用（Function Calling）的 Agent 应用，大概率经历过这样的循环：为每个 API 手写 JSON Schema，适配不同 LLM 的 tool_choice 语法，把上一版的工具集迁移到新框架时又得重写一遍。即便模型能力越来越强，但工具与模型之间的交互层始终缺乏统一协议。每次切换模型、更换编排框架，甚至只是想让同一个工具被多个 Agent 复用，都可能演变成一次小型重构。

问题出在**耦合**：工具的描述、调用方式、认证信息、状态管理散落在代码各处，没有独立的生命周期。OpenAI 的 Function Calling、Anthropic 的 Tool Use、以及开源框架各自定义的 tool 接口，虽然形态相近，却互不兼容。这不仅仅是开发效率问题，更限制了工具生态的流动——工具提供方无法“发布一次，到处运行”，Agent 开发者被迫停留在适配层打转。

## 二、MCP 到底解决了什么

Model Context Protocol（MCP）试图用一个开放的、模型无关的协议把这个中间层标准化。它定义了三个核心概念：**Resources**（可读取的数据实体，如文件、数据库记录）、**Tools**（可由模型调用的操作）和**Prompts**（预定义的交互模板）。所有的通信都通过 JSON-RPC 2.0 消息完成，支持 stdio、HTTP + SSE 等传输方式。

它解决的关键问题包括：

1. **工具与模型解耦**：一个 MCP 服务器可以被任意支持该协议的客户端使用，无论客户端背后的模型是 Claude、GPT 还是开源模型。工具不再与某个供应商绑定。
2. **语义丰富的上下文**：不仅仅是函数列表。Resources 可以暴露结构化数据，Tools 可以声明副作用和状态变化，Prompts 可以封装特定领域的交互范式，让 Agent 拥有比“一堆 function”更立体的感知。
3. **双向实时通信**：MCP 支持服务器主动通知客户端（如资源变更），适合需要长时运行、数据持续更新的自动化场景，而不只是请求-响应。
4. **安全边界天然清晰**：服务端可以精细控制哪些资源、工具对外暴露，客户端只能在授权范围内操作，避免了把整个 API 密钥和权限敞口交给大模型。

一句话概括：**MCP 把工具集成从一次性工程提升为可复用、可组合的基建层**，对 Agent 框架而言，它更像一个感知底座，而不是简单的调用管道。

## 三、快速上手：搭建一个可被 OpenClaw 消费的 MCP 服务

以 TypeScript 为例，官方 SDK `@modelcontextprotocol/sdk` 封装了大部分细节。下面是一个极简的天气工具服务器骨架：

```typescript
import { Server, StdioServerTransport } from "@modelcontextprotocol/sdk/server/index.js";
import { ListToolsRequestSchema, CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server({
  name: "weather-service",
  version: "1.0.0",
}, {
  capabilities: { tools: {} }
});

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{
    name: "get_weather",
    description: "Get current weather for a city",
    inputSchema: {
      type: "object",
      properties: {
        city: { type: "string", description: "City name" }
      },
      required: ["city"]
    }
  }]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_weather") {
    const city = request.params.arguments?.city;
    // 调用真实天气 API
    const weather = await fetchWeather(city);
    return { content: [{ type: "text", text: weather }] };
  }
  throw new Error("Unknown tool");
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

对 OpenClaw 用户来说，集成极为简单。在 OpenClaw 项目的 `mcp.json` 中配置：

```json
{
  "mcpServers": {
    "weather": {
      "command": "npx",
      "args": ["tsx", "weather-server.ts"]
    }
  }
}
```

启动 OpenClaw 后，Agent 会自动识别 `get_weather` 工具，并可按需调用。整个过程不需要修改任何 Agent 逻辑，工具就是即插即用的组件。

## 四、踩坑点与工程化建议

**传输层选择是第一个分叉口。** `stdio` 适合本地开发或同机部署，简单可靠；`HTTP+SSE` 适合远端服务，但需要处理断连重连、认证头等。如果你的 MCP 服务需要同时支持多个 Agent 实例，优先用 SSE 并做好会话隔离，否则工具调用上下文会串。

**工具 schema 的精确度决定成功率。** 模型是“读 schema 做事”的，不要吝啬 description，尤其是枚举值、格式约束。比如 `"temperature_unit": {"type": "string", "enum": ["celsius", "fahrenheit"], "description": "Must be 'celsius' or 'fahrenheit'"}` 远比省略 description 的字段调用成功率高。

**错误要结构化返回。** 当工具调用失败，不要只抛一个 HTTP 500 字符串。返回 `{ content: [{ type: "text", text: "Error: API rate limit exceeded" }], isError: true }`，这样 Agent 能读懂上下文并尝试修复或中止。

**权限不要裸奔。** 如果 MCP 服务内部调用敏感操作，必须在服务端完成鉴权，而不是依赖客户端的 tool description。例如文件系统 MCP 服务器应限制可访问路径，不暴露整个根目录。

**版本兼容性**：MCP 规范仍在快速迭代（截至 2025 年 4 月，最新为 0.5.x），使用时务必固定 SDK 版本，并在升级后重新验证工具 schema 与传输层行为。

## 五、可复用建议：站在社区肩膀上

MCP 的生态正在形成，已有大量官方和社区服务可直接使用：`@modelcontextprotocol/server-filesystem`、`@anthropic/mcp-server-github`、`@modelcontextprotocol/server-postgres` 等。先在 OpenClaw 中直接按需启动这些现成服务，几行配置就能让 Agent 接入 GitHub、数据库、本地文件。

当不得不自研时，遵循最小可行原则：先定义 Resources 暴露只读数据，再根据需要添加 Tools，避免一次性把业务逻辑全部塞进 MCP。同时为每个服务添加健康检查和调用日志，方便排障（OpenClaw 的调试模式会打印每个工具调用请求与响应）。

最后，把 MCP 服务视为独立制品进行版本管理和发布，像微服务一样对待它。这样即便 Agent 框架迭代，工具资产依然稳固。

## 六、总结

MCP 并没有发明新的 AI 能力，但它解决了工具化落地的工程核心问题：**标准化连接的最后一公里**。对于 OpenClaw 社区里大量需要编排外部 API、本地资源、数据库的自动化场景，MCP 提供了一种低摩擦、易复用的集成方式，把我们从重复的适配劳动中解放出来。当工具真正变成可组合的基础设施，Agent 的边界才会被彻底打开。

---

