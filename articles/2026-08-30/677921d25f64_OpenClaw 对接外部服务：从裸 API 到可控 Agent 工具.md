---
title: OpenClaw 对接外部服务：从裸 API 到可控 Agent 工具
feedId: 35382
source: 综合讨论
publishedAt: 2026-08-30
---

# OpenClaw 对接外部服务：从裸 API 到可控 Agent 工具

## 背景

OpenClaw 里的 Agent 进入真实业务流程后，经常要碰外部系统：查订单、建工单、拉告警、发通知。这些能力基本都通过 HTTP API 暴露。

但直接把 URL 丢给模型自由调用并不靠谱，常见结果是参数漂移、鉴权泄露、超时重试、重复提交。更稳的方式是：在 OpenClaw 中把每个外部 API 注册成明确的工具（tool/action），让模型负责意图和参数，运行时负责执行、重试、脱敏和审计。

## 常见问题

- 鉴权信息放在哪里？
- 上游错误如何翻译给模型？
- 响应太大把上下文撑爆怎么办？
- 写操作重试会不会产生重复数据？
- 限流或 429 怎么处理？

## 对接步骤

### 1. 先切边界

一个工具只做一件事，不要做 `call_api` 这种万能工具。比如：

- `get_order_status`
- `create_ticket`
- `list_recent_alerts`

边界清楚，模型才不容易乱填参数。

### 2. 声明工具

OpenClaw 里通常可以注册 HTTP 工具，声明 method、url、headers、query、body schema。鉴权信息通过环境变量注入，不要硬编码。

```yaml
tools:
  - name: create_ticket
    description: 创建工单，title 必填，priority 仅允许 low/normal/high
    method: POST
    url: https://api.example.com/v1/tickets
    headers:
      Authorization: "Bearer ${TICKET_API_KEY}"
      Accept: application/json
    body_schema:
      type: object
      required: [title, priority]
      properties:
        title: {type: string, maxLength: 100}
        priority: {enum: [low, normal, high]}
    timeout_ms: 8000
    max_retries: 2
```

### 3. 统一返回契约

上游原始 JSON 不要直接塞给模型。工具层包一层稳定结构：

```json
{"ok": true, "data": {}, "error_code": null, "request_id": "req_01"}
```

这样模型容易判断成功或失败，排障时也能用 `request_id` 查日志。

### 4. 处理错误与重试

- 4xx 不自动重试，把业务错误信息返回给模型。
- 5xx 或 429 可以做指数退避，最多 2 次。
- 创建、支付类接口必须有幂等键，比如 `client_token`，避免超时重试产生重复工单。

### 5. 小范围验证

先在 CLI 或 tool tester 里直接调用工具，确认 schema、鉴权、返回结构都符合预期，再让 Agent 在受限任务中使用。必要时增加 `dry_run` 参数。

## 踩坑点

1. **密钥写进工具描述或 prompt**：容易被日志、快照或模型输出带出。统一走环境变量或密钥存储。
2. **响应过大**：list 接口必须分页、限制 limit、裁剪字段；包装层做截断，并提示“结果已截断”。
3. **非 JSON 返回**：显式声明 `Accept: application/json`。解析失败时返回可读错误，不要抛原始堆栈。
4. **枚举和时区不一致**：上游返回 `High/Normal/Low`，内部用 `high/normal/low`，要在 schema 或 normalization 层统一。
5. **忽略限流**：429 要退避，并向模型反馈“当前被限流，请稍后重试”，避免连续重打。

## 可复用建议

- **薄封装**：查询、创建、更新分开，避免一个工具承担过多分支。
- **工具描述写清副作用**：读接口可以快速重试，写接口要谨慎；描述里写清楚“会创建真实工单”。
- **写操作加确认**：涉及生产变更时，增加 `confirm` 参数或人工确认节点。
- **可观测性**：记录 request_id、耗时、状态码、重试次数。上游没有 request_id 就自己生成一个透传。
- **先 mock 后生产**：用 sandbox 或 mock server 联调，稳定后再切生产域名。

## 总结

OpenClaw 对接外部服务的重点，不是给 Agent 打开一个任意网络出口，而是把外部 API 收编成可描述、可校验、可重试、可审计的工具。边界清晰、鉴权隔离、错误契约稳定、写操作幂等，这四点做到，Agent 才能在真实业务流程里可靠握手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/58a613fe5d3ac4c6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6fc26448febe93f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/eae8d0d9561a9961.png)

