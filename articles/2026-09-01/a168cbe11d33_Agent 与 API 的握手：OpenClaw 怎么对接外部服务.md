---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 35661
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

Agent 的优势是理解意图、拆步骤、填参数，但真正写入外部系统的动作必须由确定性的 API 调用完成。OpenClaw 场景里，外部服务可以是一个 issue tracker、审批接口、数据库 API 或公司内部网关。对接的核心不是“让模型知道有 API”，而是把模型输出约束到一个可验证、可重试、可审计的执行边界。

## 问题

直接把 REST 调用塞进提示词会有几类问题：

- 凭据暴露、难以轮换；
- 参数类型和必填项只能靠模型自觉；
- 超时、限流、非 2xx 没有统一处理；
- 响应过大时大量 token 被浪费；
- 排障时看不到请求/响应的结构化日志。

OpenClaw 的工具注册层（HTTP tool / MCP tool）适合承担这层“翻译”。

## 做法

1. 先定义工具，而不是先写 prompt。给 Agent 的 description 需要说明什么时候用、失败返回什么。参数用 JSON Schema 约束。不同版本 OpenClaw 的工具注册字段可能有差异，以你部署版本为准。

```yaml
tools:
  - name: create_ticket
    description: 在工单系统创建一条新工单。仅当用户明确要创建时使用。
    parameters:
      type: object
      required: [title, priority]
      properties:
        title: {type: string, minLength: 4, maxLength: 80}
        priority: {type: string, enum: [low, normal, high]}
        assignee: {type: string}
    http:
      method: POST
      url: "https://ticket.internal/api/v1/tickets"
      headers:
        Authorization: "Bearer ${TICKET_API_TOKEN}"
        Content-Type: "application/json"
      timeout_ms: 8000
      retry:
        max_attempts: 2
        backoff_ms: 500
        retry_on_status: [429, 502, 503]
```

2. 凭据只走环境变量或 secret 管理器，不要把 token 写进工具描述或 prompt。

3. 响应做裁剪。API 返回 100 个字段时，只保留 `id`、`status`、`url`、`error_code` 等关键信息，控制在 2–4 KB 以内，避免后续上下文膨胀。

4. 对复杂接口优先封装 MCP。如果外部系统已有 OpenAPI 3 文档，可以先导入生成工具骨架，再补鉴权和超时策略，不要手写几十个 schema。

5. 给每次调用打上 `tool_call_id`、`status`、`duration_ms`、`masked_headers`，方便回放。

## 踩坑点

- **200 不代表业务成功。** 很多系统返回 `{"code": 4001}` 和 HTTP 200，必须在 adapter 里检查业务码并转成工具错误。
- **非幂等 POST 不要盲目自动重试。** 超时重试前至少要求上游支持 idempotency key，或者改用 GET/查询确认。
- **模型会“编造”枚举值。** `enum` 和 `required` 不能只写在 description，要落在 schema 层。
- **URL 模板里的变量要做 encode。** `{{repo}}` 可能含 `/`，需要 encodeURIComponent。
- **SSRF 风险。** 允许 Agent 调用任意 URL 时，必须配置域名白名单/私有 IP 拦截。
- **报错信息不要原样回灌。** 若上游返回 HTML 或堆栈，需截断并替换成简短 error code。

## 可复用建议

- 把外部 API 分成只读和写操作。只读 GET 可缓存 10–60 秒并设置 `cache_key`，写操作禁用缓存。
- 维护一个 `service adapter` 层：统一做鉴权、超时、重试、业务码检查、响应裁剪。Agent 只看到干净的工具。
- 本地用 mock server 回放固定响应，测试 schema 是否真的能约束模型输出。
- 每个工具都返回可读的结果对象，而不是裸 JSON：`{ok: true, id: 123, url: ...}` 比把上游 body 整个塞回去稳定得多。

## 总结

Agent 与 API 的握手不是“让模型 curl”，而是把不确定性关进参数层，把执行交给确定性的 adapter。OpenClaw 里做好工具描述、schema 约束、鉴权和响应裁剪，外部服务才能真正进入可工程化的 Agent 工作流。接口对了，Agent 才会稳定；日志全了，排障才不靠猜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3f860effded44a27.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8b0d62a945b07ee0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/9c60d313f1508d5e.png)

