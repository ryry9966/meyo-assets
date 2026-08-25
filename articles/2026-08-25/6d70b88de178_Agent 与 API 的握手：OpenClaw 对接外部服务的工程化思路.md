---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化思路
feedId: 34671
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 作为 Agent 运行时，真正落地外部自动化时，绝大多数动作都需要通过外部 API 完成：创建工单、查询监控、触发部署、拉取报表。模型本身不能直接访问这些系统，它只能调用我们在 OpenClaw 中注册的工具、插件或 MCP server。

很多人第一次对接时会把问题想简单：发一个 HTTP 请求，把结果丢回给模型。但实际跑起来后，问题往往不在 API 调不通，而在 Agent 与 API 之间的“握手”不稳定——参数乱编、响应过大、错误不可读、重试导致重复提交。这些才是真正消耗工程时间的地方。

## 问题

常见的坑集中在几个点：

- **参数幻觉**：模型会自行发明字段值、枚举值或日期格式。
- **上下文爆炸**：API 返回几十 KB 的 JSON，模型很快丢失重点。
- **凭证散落**：token 写在工具描述、环境变量硬编码或日志里。
- **超时与重试**：上游慢一点，Agent 可能会重复调用工具。
- **错误不可读**：把 500 的 HTML 或堆栈直接返回给模型，模型无法做出正确决策。

## 做法 / 步骤

在 OpenClaw 里对接外部服务，我更推荐优先使用 MCP server，而不是直接写一个巨大的 HTTP 工具。原因是 MCP server 可以独立调试、复用，并且 OpenClaw 加载后会自动注册工具，不用改主流程。

### 1. 先注册 MCP server

以一个内部工单系统为例，可以在 OpenClaw 的 MCP 配置里挂一个本地 server：

```yaml
mcp_servers:
  ticket:
    command: node
    args: ["./mcp-ticket-server/index.js"]
    env:
      TICKET_API_BASE: "https://ticket.internal/api"
      TICKET_TOKEN: "${TICKET_TOKEN}"
```

`TICKET_TOKEN` 从环境变量注入，不要写死在配置文件里。

### 2. 定义工具的输入 schema

工具参数要尽量明确，枚举不要给模型自由发挥的空间。示例：

```ts
const ticketSchema = z.object({
  title: z.string().min(2).describe("工单标题，简短概括问题"),
  priority: z.enum(["low", "medium", "high"]).describe("优先级，用户未说明时默认 medium"),
  body: z.string().describe("工单详细描述")
});
```

字段描述不只是给开发者看，更是给模型看的。描述越具体，误调用概率越低。

### 3. 封装 API 客户端并控制返回体

在 MCP server 里调用真实 API，但不要直接把上游响应原样返回。只抽取模型继续决策需要的字段：

```ts
const data = await res.json();
return {
  content: [{
    type: "text",
    text: JSON.stringify({
      id: data.id,
      status: data.status,
      url: data.url
    })
  }]
};
```

返回体控制在 2KB 以内，多余字段对 Agent 多数情况下是噪声。

### 4. 错误要翻译

上游返回非 2xx 时，不要返回 HTML 或堆栈。转成结构化错误文本，例如：

`TICKET_API_UNAVAILABLE: upstream returned 503`

这样模型至少知道是上游不可用，而不是继续盲目重试。

### 5. 测试

先用 MCP inspector 或命令行直接调用 server 的工具，确认参数、返回、错误分支都正常，再让 OpenClaw 加载并跑真实任务。

## 踩坑点

1. **超时与重复提交**  
   外部 API 慢时，Agent 端可能会认为工具失败并再次发起调用。建议在 API 客户端层设置 8–12 秒超时，并让重试逻辑落在客户端层，而不是依赖模型重试。创建类接口必须支持幂等键，例如 `idempotency-key` 或 `client-request-id` 头。

2. **列表接口不要默认全量拉取**  
   如果 API 返回列表，务必加 `limit`，默认 10 或 20，并提供分页参数。一次拉全量会把上下文直接撑爆。

3. **凭证泄漏**  
   不要在任何工具描述、系统提示或调试日志里打印 token。MCP server 的好处是凭证隔离在 server 进程内，模型只看到工具接口。

4. **模型误调用工具**  
   工具描述里要写触发条件，例如“仅当用户明确要求创建工单时调用，不要用于查询或闲聊”。否则模型可能把查询动作也当成创建动作。

5. **上下文累积**  
   多轮工具调用后，旧结果仍然占用上下文。可以给工具返回设置合理的保留策略，或让 OpenClaw 端裁剪历史，避免 Agent 基于过时信息继续操作。

## 可复用建议

- **工具做小**：一个 tool 只做一件事，避免一个接口承担创建、查询、更新多个语义。
- **Schema 先行**：先花时间把字段、枚举、默认值定清楚，再写实现。
- **加 request_id**：每次 API 调用生成唯一 ID，贯穿上游日志与 Agent tool call，方便排障。
- **限流与熔断**：给外部 API 设置独立限流，避免 Agent 循环调用把第三方服务打爆。
- **敏感操作确认**：删除、发布、支付等动作在工具层或 OpenClaw 侧加用户确认。
- **先 MCP，后直连**：需要长期维护的服务用 MCP server 封装；临时脚本可以用简单 HTTP tool，但不要把原始 REST 接口直接暴露给模型。

## 总结

外部 API 对接不是“把 HTTP 请求包一层”这么简单，它本质上是把不可靠、冗长、错误各异的外部服务，翻译成 Agent 能稳定理解、稳定调用的工具接口。OpenClaw 里优先通过 MCP server 做凭证隔离、schema 约束、响应裁剪和错误翻译，再做一次完整的 tool call 链路测试，能避免大部分早期返工。稳定的握手，来自稳定的契约，而不是让模型自己适应混乱的 API。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/b47aa98676e90c4f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ecb33255a8516729.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/12c54d61c45693e2.png)

