---
title: OpenClaw 对接外部服务：从 HTTP 到 MCP 的工程化握手
feedId: 34257
source: 综合讨论
publishedAt: 2026-08-23
---

# Agent 与 API 的握手：OpenClaw 怎么对接外部服务

## 背景

OpenClaw 里的 Agent 要做成可用的自动化，迟早要碰外部服务：查订单、发消息、建工单、取数据。但 LLM 本身不处理网络、鉴权、限流和脏数据，这些必须由框架层或工具层接住。

一个问题反复出现：外部 API 是给人类开发者写的，不是给 Agent 写的。字段深、错误码不统一、文档冗长、副作用明显。直接把 API 暴露给模型，轻则 token 浪费，重则误操作。

这篇讲我在 OpenClaw 里对接外部服务时踩过的点，以及一套相对稳的工程化做法。

## 问题：手伸得太长

很多第一次对接的人会把整个 API 的 OpenAPI spec 塞进工具 description，或者把十几个 endpoint 做成一个大工具，参数全部可选。结果模型乱填参数，甚至把创建订单当成查询订单。

核心矛盾：Agent 需要的是小粒度、契约清晰、副作用明确的工具；而外部 API 通常是大而全、面向人类前端设计。握手的关键不是“让模型会调 API”，而是把 API 翻译成模型能安全调用的形状。

## 做法：薄适配层 + 工具契约

我的几步固定动作：

**1. 定义工具边界，而不是搬运 API**

把外部服务拆成几个小工具，每个只做一类操作。例如：

- `search_orders(date_range, status)` — 只读，无副作用
- `create_order(customer_id, sku, quantity, idempotency_key)` — 写操作，强制幂等键
- `get_api_health()` — 诊断用

不要一个工具支持所有动作。拆分后模型更容易选对。

**2. OpenClaw 中定义工具 schema**

如果用 MCP，就写一个轻量 MCP server，暴露 `tools`。OpenClaw 侧接入 MCP server 即可，用 JSON Schema 描述参数。

```json
{
  "name": "search_orders",
  "description": "查询最近 N 天的订单，只读。返回订单号、状态、金额、创建时间。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "days": { "type": "integer", "minimum": 1, "maximum": 90 },
      "status": { "type": "string", "enum": ["pending", "paid", "cancelled"] }
    },
    "required": ["days"]
  }
}
```

要点：`description` 里写清楚“只读”“副作用”“返回格式”，不要写 API 路径。

**3. 适配层统一返回信封**

第三方响应往往有嵌套、垃圾字段、错误页。适配层里裁剪成统一格式：

```json
{
  "ok": true,
  "data": { "orders": [...] },
  "error": null,
  "request_id": "a1b2c3"
}
```

模型看到 `ok: false` 和简洁的 `error` 时，能直接判断失败原因，不需要读一堆 HTML。

**4. 鉴权与敏感信息**

API key 放环境变量，适配层启动时读取。工具 description 里不要出现 key。MCP server 不要打日志把 `Authorization` 头打出来。

**5. 超时、重试、限流**

外部 API 常见 429、5xx。适配层要做有限重试：

- 超时：connect 3s，read 10s（根据 API 调整）
- 重试：只对 429、502、503、504，最多 2 次，指数退避 + jitter
- 429 时读取 `Retry-After`，尊重服务端节奏

不要在 Agent 层重试，重试属于适配层。

## 踩坑点

**Schema 写太宽**：`additionalProperties: true` 加可选参数一大堆，模型会自己发明参数。字段限制要严格，`required` 和 `enum` 能用就用。

**Description 塞 API 文档**：几千 token 的 description 会抢走上下文，模型反而更迷茫。只写“做什么、限制、返回什么”。

**错误页当数据返回**：很多第三方在 5xx 时返回 HTML，适配层不拦截，模型会把 `<html>` 当数据。一定要在适配层检测状态码和内容类型，非 JSON 就返回统一错误。

**缺少幂等键**：写操作没幂等键，Agent 重试或用户多问一次，就重复创建。适配层应该强制传入 `idempotency_key`，没有就拒绝。

**忽视速率限制**：批量任务触发 429，模型不知道怎么处理，就开始瞎编结果。适配层要把 429 翻译成明确的“稍后重试，已排队”或直接失败让上层调度。

## 可复用建议

- 外部服务对接采用“薄适配层”模式：OpenClaw 调适配层工具，适配层调外部 API。
- 返回统一信封：`{ ok, data, error, request_id }`。
- description 里写副作用：只读 / 会创建 / 会扣款 / 会发消息。
- 复杂外部服务优先做成 MCP server，而不是在 OpenClaw 里写裸 HTTP 工具。
- 工具调用日志要保留 request_id，方便排障。
- 每个工具至少写三个测试：正常返回、第三方超时、第三方返回错误结构。用 CLI 或 playground 先测，再交给 Agent。

## 总结

“Agent 与 API 的握手”不是一句口号，而是契约设计。OpenClaw 的 MCP/工具层很适合做这个翻译层，前提是你愿意花时间裁剪 schema、统一错误、处理重试和幂等。

把这层做好之后，Agent 不再需要理解外部服务的混乱，它只需要面对几个可信的小工具。外部 API 终于成了被安全握住的“手”，而不是随时会抓伤模型的爪子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/61e846af896bb000.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/44c5a6f392ab809a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ffb050ef8de1502c.png)

