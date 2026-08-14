---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 33082
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

OpenClaw 里接入外部 API，常见路径有两种：走 MCP server，或直接写 OpenClaw 插件/tool。无论哪种，本质都是把“模型的不确定输出”转成“外部服务的确定调用”。

很多问题不是接口调不通，而是握手不稳定：字段类型错、超时不重试、错误信息直接漏给用户、返回体太大把上下文打爆。我们更需要一套可复用的工程化接法，而不是一次性脚本。

## 问题

Agent 和 API 的契约天然不对称：

- Agent 擅长理解意图，但会自由发挥参数；
- API 要求严格类型、枚举、鉴权、超时；
- 外部服务失败时，Agent 容易把原始错误当成事实继续推理；
- 重试不当会造成限流风暴。

所以在 OpenClaw 里对接外部服务，核心不是“调用成功一次”，而是让调用边界清晰、失败可观测、行为可预期。

## 做法/步骤

以 OpenClaw 自定义 tool 对接一个订单查询 API 为例。

**1. 先定义严格 JSON Schema，不要裸调**

工具描述和参数 schema 要写得足够死，避免模型乱填。例如：

```yaml
name: order_lookup
description: >
  Query a single order by order ID. Read-only, idempotent.
  Use when user asks about order status, logistics, or refund state.
parameters:
  type: object
  properties:
    order_id:
      type: string
      pattern: "^[A-Z0-9]{8,32}$"
      description: "Order ID from ERP, not the display number."
  required: [order_id]
```

重点：枚举用 string literal；布尔参数加 `type: boolean`，handler 里容错字符串 `"true"`；时间参数统一用 ISO 8601。

**2. 在 handler 里封闭鉴权、超时、重试**

外部 API 的密钥不要进 prompt，也不要写死在插件代码里。统一从环境变量或 secret store 读取。handler 内层建议固定：

```text
timeout: 5s 或 10s
retry: 最多 2 次，只对 429/5xx 重试
backoff: 指数退避
```

不要无限重试，否则一个外部服务抖动就会把 OpenClaw 拖死。

**3. 响应裁剪后再返回给模型**

外部 API 经常返回巨量字段。不要让模型自己找重点，handler 先裁剪：

```json
{
  "ok": true,
  "order": {
    "id": "A12345678",
    "status": "shipped",
    "carrier": "SF",
    "eta": "2025-01-10"
  }
}
```

只保留后续推理需要的字段。列表查询建议在 handler 内做分页循环或 `limit` 限制，不要把分页 cursor 暴露给模型处理。

**4. 错误归一化**

外部服务返回的错误不要原样抛给模型。统一成：

```json
{
  "ok": false,
  "error_code": "ORDER_NOT_FOUND",
  "user_message": "没有查到该订单，请确认订单号是否正确。",
  "retryable": false
}
```

这样模型既能理解，又不会把内部堆栈、API key、网关地址等细节带出来。

**5. 回放测试**

接完先别直接上真实任务。用固定输入集跑几轮，检查参数生成是否稳定、超时分支是否生效、错误返回是否符合预期。有 request_id 的 API，务必在日志里记录，方便和外部服务对账。

## 踩坑点

- **模型把非必填参数也填了**：有些 API 对空字符串很敏感，schema 里能限制就限制，handler 里做空值过滤。
- **429 重试风暴**：业务低峰一次限流，如果 tool 层无限重试，会放大故障。只对明确可恢复错误做有限重试。
- **时间/时区错乱**：外部服务若要求时间戳，明确单位是秒还是毫秒，handler 里统一转换，不要让模型算。
- **写操作没有确认**：退款、取消、发送等有副作用操作，建议加 human-in-the-loop，或者至少 tool description 标注 `side_effect: true`，并在 handler 里做 dry-run 校验。
- **MCP server 掉线无感**：如果走 MCP，需要给 server 做健康检查，失败时 agent 应降级提示“外部服务暂不可用”，而不是继续编造数据。

## 可复用建议

1. **只读优先**：先接查询、搜索、列表类 API，再接写操作。
2. **一个 tool 只做一件事**：避免一个工具既查订单又改库存，描述会变得模糊。
3. **返回结构保持稳定**：`{ok, data|error}` 比自由格式更适合模型解析。
4. **请求 ID 贯穿日志**：OpenClaw tool 日志、外部 API 日志、用户会话 ID 三者能对上，排查效率高很多。
5. **给工具打标签**：read-only / idempotent / side_effect，方便后续治理。

## 总结

OpenClaw 对接外部服务，不是简单调通一个 REST API，而是把外部能力装进 Agent 的稳定边界里。关键是：严格 schema、封闭式 handler、归一化错误、有限重试、可观测日志。先把握手做稳，再让 Agent 发挥推理能力，线上才不会出现“接口是通的，但 Agent 用不起来”的尴尬。

---

