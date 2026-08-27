---
title: OpenClaw 对接外部服务的工程化姿势：从工具注册到 MCP 的避坑笔记
feedId: 34970
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

Agent 不能只靠模型推理。查订单、发通知、操作工单，都需要调用外部 API。OpenClaw 提供两种主要接入方式：内置 HTTP 工具注册和 MCP server。前者适合简单 REST API，后者适合已有标准工具服务的场景。真正容易出问题的不是“能不能调通”，而是模型会不会用、参数准不准、出错后能不能恢复。

## 问题

把外部 API 直接丢给 Agent，常见三类问题：工具描述模糊导致误调用；参数 schema 太宽导致传参混乱；API 报错被直接抛给模型，Agent 要么反复重试，要么给出错误结论。此外还有鉴权泄露、响应体积过大、超时卡死等工程问题。

## 做法/步骤

### 1. 定义工具边界

一个工具只做一件事。比如 `query_order` 只查订单，不取消、不改地址。触发条件写清楚：“当用户询问订单状态、物流进度时使用”。

### 2. 注册 HTTP 工具

在 OpenClaw 工具配置里写明 name、description、parameters、endpoint、method、headers、timeout。示例：

```yaml
tools:
  - name: query_order
    description: Query order status by order ID. Use when user asks about order delivery or status.
    parameters:
      type: object
      properties:
        order_id:
          type: string
          description: Order ID, format ORD-2024-xxxx
      required: [order_id]
    endpoint: https://api.internal.example.com/orders/{order_id}
    method: GET
    headers:
      Authorization: Bearer ${ENV:ORDER_API_TOKEN}
    timeout_ms: 5000
    retry:
      max_attempts: 2
      backoff_ms: 500
```

路径参数用 `{order_id}` 占位，鉴权头通过 `${ENV:...}` 注入，不要硬编码。

### 3. 错误语义映射

不要直接返回 HTTP 500 或原始 JSON 错误。配置映射：404 → “订单不存在，请核对单号”；401 → “鉴权失败，不要重试”；429 → “限流，稍后重试”。这样模型能根据错误信息决定下一步。

### 4. 用 MCP 接入

如果外部服务已经提供 MCP server，优先用 MCP。OpenClaw 只需配置连接信息，工具发现和参数 schema 由 server 提供，减少手工同步。注意区分 stdio 和 SSE 传输，本地敏感信息不要随配置提交。

### 5. 本地联调

先用 curl 验证 API 可达，再注册到 OpenClaw。写一个 smoke test：遍历所有工具定义，发一次最小合法请求，检查返回字段是否与描述一致。

## 踩坑点

- **描述写成给人看的**：例如“获取订单信息”不够，要写“根据 order_id 返回订单状态、物流单号、预计送达时间。当用户问‘我的订单到哪了’时必须调用”。
- **参数 schema 太宽**：string 字段不给格式或枚举，模型可能传空字符串、错误类型或超长内容。尽量用 enum、pattern、minLength/maxLength 约束。
- **超时设置不合理**：太短正常慢请求失败，太长 Agent 卡住。读接口 3-8 秒，写接口 5-10 秒。
- **写操作自动重试**：查询可重试 1-2 次，但创建订单、扣款、发送通知等写操作不要自动重试，否则容易重复执行。
- **鉴权信息泄露**：header 里直接写 token 会出现在日志、工具定义或模型上下文中。用环境变量注入，并确保日志脱敏。
- **响应体积过大**：API 返回几 MB JSON，模型上下文爆掉。在工具层做字段裁剪，只返回模型需要的关键字段。

## 可复用建议

- 工具契约四要素：单一职责、明确触发条件、窄参数 schema、错误语义映射。
- 在外网 API 和 OpenClaw 之间加一层薄网关，统一处理鉴权、超时、重试、字段裁剪、审计日志。OpenClaw 只对接网关，不直接暴露原始 API。
- 每次修改工具定义后跑 smoke test，防止 schema 破坏。
- 观察 OpenClaw 的工具调试日志，看模型实际生成什么参数、API 返回什么，再迭代描述。
- 对不稳定 API 加缓存或降级返回，避免 Agent 陷入反复重试。

## 总结

Agent 对接外部服务，本质上是在模型和 API 之间建立一份“工程契约”。OpenClaw 的工具注册和 MCP 降低了接入成本，但真正决定稳定性的是错误处理、参数约束和上下文控制。先把 API 当给人用的薄封装，再让模型去调用，成功率会高很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/fc2eab6a13383aaa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/58f4631c45b93d71.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b13250b5be2b95c2.png)

