---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 33998
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景：为什么“对接外部服务”是 Agent 的分水岭

OpenClaw 能跑通本地对话、能调用内置工具，只是第一步。真正进入生产环境，往往需要让 Agent 和已有的业务系统、第三方 API、内部平台打交道：查订单、建工单、发通知、拉报表。这个“握手”过程并不神秘，但很容易做成“能跑通一次，换个人用就崩”的demo。

这篇文章不讨论通用概念，只围绕一个具体问题：**在 OpenClaw 里，怎么把外部 API 接得稳、可复用、不烧 token**。

## 问题：Agent 调用 API 不是“加个 URL”那么简单

很多人第一反应是：写个 tool，里面用 `fetch` 请求一下，把结果返回给模型。结果遇到三类典型故障：

1. **上下文爆炸**：外部 API 返回 40KB JSON，模型还没开始总结，context 已经吃掉一半。
2. **参数漂移**：模型传参时字段名对不上、空值乱传、时间格式不统一。
3. **鉴权与重试不可控**：密钥写死在代码里，或者 API 超时导致整个 Agent 卡住。

核心矛盾是：**外部 API 的返回结构是为“人/后端系统”设计的，不是为“模型上下文”设计的**。所以对接的第一步不是写代码，而是做“边界翻译”。

## 做法：用薄封装 + MCP/插件的方式接入

下面按一个通用外部服务（假设是“工单系统 API”）说明步骤。OpenClaw 里推荐通过 **MCP server 或自定义插件** 接入，而不是直接在提示词里塞 API 文档。

### 1. 先定义最小工具面

不要一次暴露 20 个接口。先挑两个高频动作：

- `search_tickets`：按关键词/状态查工单
- `get_ticket_detail`：查看单个工单详情

每个工具只做一件事，参数控制在 3 个以内。例如：

```json
{
  "name": "search_tickets",
  "description": "Search support tickets by status and keyword. Use status 'open' or 'closed'.",
  "parameters": {
    "type": "object",
    "properties": {
      "status": { "type": "string", "enum": ["open", "closed"] },
      "keyword": { "type": "string", "description": "match in title or description" }
    },
    "required": ["status"]
  }
}
```

### 2. 实现 handler：鉴权、超时、裁剪

在 MCP server / 插件代码里，不要直接返回原始响应。做三层处理：

- **鉴权**：API Key 从环境变量读，例如 `TICKET_API_KEY`。
- **超时与重试**：用 `AbortSignal.timeout(8000)`，外部 API 失败只重试一次，且只对 `5xx` 或网络错误重试。
- **响应裁剪**：只返回模型需要的字段。例如列表接口最多返回 10 条，每条只保留 `id`、`title`、`status`、`updated_at`。

示例伪代码：

```ts
const res = await fetch(`${API_BASE}/tickets?status=${status}&q=${keyword}`, {
  headers: { Authorization: `Bearer ${env.TICKET_API_KEY}` },
  signal: AbortSignal.timeout(8000)
});
const data = await res.json();
return {
  total: data.total,
  items: data.items.slice(0, 10).map(t => ({
    id: t.id,
    title: t.title,
    status: t.status,
    updated_at: t.updated_at
  }))
};
```

### 3. 注册到 OpenClaw

如果以 MCP server 方式接入，在 OpenClaw 的 MCP 配置里加上一条：

```json
{
  "mcpServers": {
    "ticket-service": {
      "command": "node",
      "args": ["/path/to/ticket-mcp/build/index.js"],
      "env": { "TICKET_API_KEY": "${TICKET_API_KEY}" }
    }
  }
}
```

插件方式则把工具函数导出，并声明 JSON Schema。两种方式都能让 OpenClaw 在运行时发现工具，不需要手动改主提示词。

## 踩坑点

### 坑 1：description 写得太随意

模型是不会看代码注释的，它只看工具描述和参数 Schema。描述里不要写“获取工单信息”这种废话，要写清楚**什么时候用、返回什么、有哪些约束**。否则模型会拿错工具或漏传参数。

### 坑 2：错误信息不可读

外部 API 返回 `{"code": 40301, "message": "permission denied"}` 时，不要直接把这个抛给模型。捕获后改成：

```
Ticket API permission denied. Please check that the API key has read access to the 'support' project.
```

模型才能自我纠正，而不是重复调用。

### 坑 3：把分页、排序全交给模型

模型不擅长做分页计算。工具内部固定 `limit=10`，排序固定 `updated_at desc`。如果用户真的需要更多，再加一个 `offset` 参数，而不是让模型自己拼 `page=3&size=50`。

### 坑 4：密钥写进 MCP 配置并提交

用环境变量占位符，部署时再注入。不要为了“方便”把真实密钥写进配置文件或插件代码。

## 可复用建议

- **一个外部服务，一个薄 MCP server**，不要在一个大插件里堆多个服务。
- **给每个工具写 2-3 个 golden case**：给定自然语言问题，验证 tool call 参数是否正确。这比只测 API 通不通重要得多。
- **记录 OpenClaw 的 tool call trace**：看模型实际传了什么参数，比看日志里的报错更有用。
- **为只读接口加本地缓存**：例如查询客户信息，60 秒内相同参数直接返回缓存。能显著降低外部 API 压力，也降低延迟。
- **限制并发调用**：如果 OpenClaw 支持并行工具调用，给外部服务加一个简单的信号量，避免突发流量把第三方打挂。

## 总结

OpenClaw 对接外部服务，本质是**做一层“模型友好”的 API 适配**。不要让模型直接接触原始 API 的复杂结构，而是通过最小工具面、严格 Schema、响应裁剪、明确错误信息，把不确定性挡在 Agent 之外。

这个工作没有太多炫技成分，但能让 Agent 从“偶尔能用”变成“团队敢用”。先把一个外部服务接稳，再复制模式到第二个、第三个，就是最务实的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ec3994309f135690.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ce99956e08150fdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/0d13cad59214f7c7.png)

