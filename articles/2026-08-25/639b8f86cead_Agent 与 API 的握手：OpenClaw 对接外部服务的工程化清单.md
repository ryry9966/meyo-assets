---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化清单
feedId: 34673
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 不是只会聊天的 Agent，它需要和外部世界握手：查天气、发消息、建工单、读数据库、调用企业内部 RPC。握手的主要形式就是 API。OpenClaw 的插件机制、Tool 注册和 MCP 支持都提供了接入点，但真正把外部 API 接稳，关键不在“能调通”，而在“把不可控的开放 HTTP 变成可控工具”。

## 问题

最初容易把外部 API 直接交给模型：在系统提示里写“你可以调用 https://api.example.com”，或者让模型自己拼 curl。这样很快会暴露几个问题：鉴权信息容易泄漏；参数结构不稳定；超时与错误没有处理；响应过大撑爆上下文；分页和限流没人管。结果是演示能跑，生产一用就崩。

## 做法 / 步骤

1. 选对接入层。优先把外部 API 封装成 OpenClaw 的本地 tool 或 plugin，而不是让模型直接发 HTTP。已有 MCP server 的外部服务，用 OpenClaw 的 MCP 配置挂载；临时脚本用 command tool 包一层。

2. 定义工具描述和参数 schema。每个 tool 只暴露必要参数，不要为了“灵活”把所有 query 参数都开放。例如：

```json
{
  "name": "get_order_status",
  "description": "查询单个订单状态，返回订单号、状态、更新时间。",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "订单号" }
    },
    "required": ["order_id"]
  }
}
```

描述里写清楚返回内容和限制，比如“只返回 JSON 摘要，不返回原始响应”。

3. 封装客户端。base_url、timeout、headers、auth 放代码或环境变量，绝不放 prompt。给每个外部 API 建一个薄 client，统一处理超时、重试、错误转换。

4. 裁剪响应。外部 API 经常返回几十 KB 的 JSON 或 HTML。在 tool 返回前裁剪到关键字段，控制 token 消耗。分页接口要么自动合并，要么返回 cursor 让 Agent 决定是否继续。

5. 错误结构化。把 HTTP 错误转成 Agent 能理解的消息，例如 `{"error": "upstream_timeout", "retryable": true, "detail": "订单服务 3s 未响应"}`，不要返回 stack trace 或 HTML。

6. 记录调用日志。记录 tool 名、参数、耗时、状态码、返回大小，但避免记录 Authorization 等敏感 header。

## 踩坑点

- API key 写在工具描述或系统提示里，调试时被模型复述出来，或者日志外泄。
- schema 放太宽，模型把不存在的参数填进去，外部 API 直接 400。
- 默认 timeout 太短，查询类接口慢 1 秒就被判失败，Agent 开始重试，加重后端压力。
- 不处理 429。没有指数退避，Agent 在限流后疯狂重试，触发更严格的封禁。
- 分页只取第一页。工具返回“成功”，实际数据只有 20 条里的前 5 条。
- 返回体不裁剪。一个 30KB 的 JSON 塞进上下文，后续推理变慢、成本增加，还容易丢关键信息。

## 可复用建议

- 做一个 `createApiTool` 工厂，集中处理鉴权、超时、响应裁剪、错误转换，避免每个工具重复实现。
- 接新 API 前先写一个 smoke test，用真实响应校验 schema，别等 Agent 踩错才发现字段对不上。
- 工具描述里给一个最小调用示例，例如 `get_order_status("SO123456")`，帮助模型理解调用方式。
- 对幂等 GET 加短 TTL 缓存；写操作提供干跑或确认参数，尤其是删除、支付类接口。
- 建一个外部 API 接入清单：健康检查地址、超时阈值、重试次数、降级策略、负责人。

## 总结

Agent 与 API 的握手，不是把 URL 丢给模型，而是在 OpenClaw 的工具边界上做一层薄封装：参数收窄、鉴权隐藏、响应裁剪、错误结构化。做得好的外部服务接入，会让 Agent 觉得“这就是一个稳定函数”，而不是“一个随时会超时的网页”。这条边界，就是 OpenClaw 对接外部服务时最值得投入的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/79355138a12d63df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/7c130bacf1ed655d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/a4c43a2ca368c30c.png)

