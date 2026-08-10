---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 32455
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：工具调用的“巴别塔”乱象

如果你在搭建 OpenClaw 这类智能体时会发现，为模型赋予工具能力早已不是新鲜事。从 OpenAI 的函数调用（Function Calling），到 LangChain 的 Tool 抽象、Semantic Kernel 的插件模型，每家都有自己的一套约定。问题来了：当你希望同一个工具能被 Claude、Gemini、Llama 或内部模型复用时，往往需要为每个框架重新封装一遍，有时连参数的 JSON Schema 都要调整格式。更麻烦的是，不同模型供应商对“上下文”的理解差异很大，有的只传一段 system prompt，有的要求在每次请求时动态注入工具列表，这让工具开发者和模型消费者之间耦合极深。

这种碎片化正是 Anthropic 推出 MCP（Model Context Protocol）的背景。MCP 试图为模型-工具交互提供一个统一的、厂商无关的应用层协议，就像 USB 针对外设，HTTP 针对 Web 服务一样。它把模型视为客户端（MCP Client），工具、数据源、提示模板等能力提供方视为服务器（MCP Server），双方通过标准消息交换上下文。

## MCP 实际解决了什么

**1. 跨模型的工具复用难题**
过去要为 Claude 写一个搜索引擎工具，需要遵循其 tool_use 格式；给 OpenAI 就得按 function 结构重新组织。MCP 规定了一套标准 JSON-RPC 2.0 消息，工具定义用 `tools/list` 返回，调用用 `tools/call`，无论背后是什么模型，只要它集成了 MCP 客户端，就能无缝接入任何 MCP 服务器。OpenClaw 等支持 MCP 的项目可以直接加载社区现成的服务器，例如文件系统、数据库查询、天气 API 等，省去大量适配工作。

**2. 动态上下文注入**
传统工具调用往往只关注“调用 - 返回结果”这一条链路。MCP 将上下文扩展为三个核心概念：资源（Resources）、工具（Tools）和提示模板（Prompts）。资源可以是一段只读数据（如文件内容、数据库记录），模型可以主动请求；工具是允许模型执行的函数；提示模板则帮助自动化构建对话。更关键的是，服务器可以通过 `notifications/resources/updated` 主动通知客户端上下文发生变化，模型可以据此重新决策。这对长时间运行的 Agent 非常重要，例如监控文件变化并持续分析。

**3. 安全与隔离**
MCP 将工具执行放在用户可控的服务器进程中，而非直接暴露在模型沙箱里。权限控制、速率限制、日志审计都可由服务器实现。传输层支持标准输入/输出（stdlib）和 HTTP + SSE 两种模式，前者适合本地进程，后者支持远程部署。这种分层让生产环境的安全审核更可控。

## 工程实践：从 0 搭建一个 MCP 天气工具

假设我们想给 OpenClaw 智能体接入一个实时天气查询能力，这里展示基于 TypeScript 的最小化实现。

**步骤 1：初始化 MCP Server**
使用官方 `@modelcontextprotocol/sdk`，创建服务器并声明一个天气工具：

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "weather-server",
  version: "1.0.0",
});

server.tool(
  "get_weather",
  "Get current weather for a city",
  {
    city: { type: "string", description: "City name, e.g. Beijing" },
  },
  async ({ city }) => {
    const weather = await fetchWeatherApi(city); // 伪代码
    return {
      content: [{ type: "text", text: weather.summary }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**步骤 2：客户端连接（以 OpenClaw 视角）**
OpenClaw 等 Agent 框架内置了 MCP 客户端能力，通过配置服务器启动命令即可加载：

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["dist/weather-server.js"]
    }
  }
}
```

启动后，客户端会自动完成 `initialize` 握手，通过 `tools/list` 发现可用工具。当模型决定调用 `get_weather` 时，客户端发送 `tools/call` 请求，服务器返回结构化结果。

## 踩坑点与真实教训

- **stdio 断连后状态丢失**：如果 MCP 服务器进程崩溃，客户端连接直接断开，当前会话的所有工具状态都会丢失。生产环境建议配合进程守护（如 systemd）和健康检查。远程部署使用 SSE 模式时要处理重连和消息去重。

- **工具定义过于宽泛**：MCP 允许任意 JSON Schema 定义参数，但部分模型的函数调用层对嵌套对象、`oneOf` 等高级 Schema 支持并不完善。实践中应尽量扁平化参数，优先使用 `type: "string"` 和 `enum`，避免过度依赖复杂验证。

- **并发与锁**：`tools/call` 是异步执行的，但服务器端需要注意共享资源竞争。例如文件系统服务器同时被多个模型读写，务必实现或借助数据库事务。

- **权限提示不够显式**：默认所有工具对模型可见，但业务上可能需要用户授权。MCP 本身没有内置请求用户许可的交互协议，需要额外在服务器中嵌入 OAuth 或通过资源通知模拟。目前社区在讨论 `asks` 类型的交互，但尚未标准化。

## 可复用建议

1. **优先复用社区服务器**：`https://github.com/modelcontextprotocol/servers` 已有覆盖文件系统、Postgres、GitHub 等常用能力的服务器。直接配置比自研更稳定。

2. **用命名空间区分工具来源**：如果有多个 MCP 服务器提供类似能力，建议在工具名前加前缀（如 `weather_get`），避免混淆。

3. **本地开发用 stdio，生产用 SSE**：stdio 调试简单，但远程部署必须切换到 HTTP+SSE，并注意 CORS 和 TLS 终结。

4. **为服务器加上版本和日志**：MCP 的 `initialize` 消息会交换服务器版本，合理的版本策略利于客户端做兼容判断。所有工具调用应记录结构化日志，便于审计和调试。

5. **不要忽视资源概念**：工具不是唯一的能力载体。如果有一些只读上下文（如手册、规范文档），用资源暴露更合适，模型可通过 `resources/read` 按需加载，减少 Prompt 长度浪费。

## 总结

MCP 并非要取代现有 Agent 框架，而是提供一个标准化的上下文接入层。对于 OpenClaw 用户来说，这意味着工具市场可以从“每个模型各自造轮子”向“一次编写，随处运行”过渡。当然，协议还在早期迭代（目前最新为 2024-11-05 版本），缺少成熟的认证、流控和工具组合机制。但作为工程团队，现在引入 MCP 可以显着降低未来切换模型或扩展 Agent 能力的沉没成本。与其继续维护一堆胶水代码，不如向上游标准靠拢。

---

