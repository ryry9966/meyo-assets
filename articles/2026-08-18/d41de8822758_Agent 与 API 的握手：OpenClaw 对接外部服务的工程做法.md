---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程做法
feedId: 33764
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

OpenClaw 里的 Agent 最终要落到真实系统：查订单、建工单、重启服务、同步数据。模型本身不该直接裸调 HTTP，否则每次调用都变成一次不可控的临时脚本。更稳的做法是让外部服务通过一个受约束的“握手层”暴露给 Agent，OpenClaw 只负责调用工具，不负责理解 HTTP 细节。

这个握手层通常用一个 MCP server 或内部插件来实现。下面是我在 OpenClaw-CN 社区实践后沉淀下来的一套做法，偏工程向，不追求花哨。

## 问题

直接让 Agent 对接外部 API，常见四类问题：

1. **认证泄漏**：API key 被写进 prompt、工具描述或 trace，审计时才发现。
2. **超时与重试失控**：模型或中间层不统一处理，一个慢接口能把整个任务拖死。
3. **错误不可读**：上游返回 HTML 错误页或堆栈，模型被灌一堆噪声，还容易误判。
4. **schema 漂移**：外部接口一改，Agent 生成的参数就开始乱，调用失败率升高。

这些问题不是“换个更强的模型”能解决的，必须在握手层做约束。

## 做法 / 步骤

### 1. 用 MCP server 封装外部服务

在 OpenClaw 里优先用 MCP server 暴露外部能力。一个外部系统对应一个 MCP server，不要做一个“万能 HTTP 工具”。例如：

```yaml
# openclaw mcp 配置片段
mcp_servers:
  ticket:
    command: node
    args: ["/opt/mcp/ticket-server.js"]
    env:
      TICKET_API_KEY: ${TICKET_API_KEY}
```

工具只暴露最小动作：`get_order`、`create_incident`、`restart_service`。不要让模型直接填 URL、method、body。

### 2. 统一定义工具 schema

每个工具的参数用 JSON Schema 约束，枚举值写清楚：

```json
{
  "name": "create_incident",
  "parameters": {
    "type": "object",
    "properties": {
      "priority": { "type": "string", "enum": ["low", "medium", "high"] },
      "title": { "type": "string", "maxLength": 120 },
      "idempotency_key": { "type": "string", "description": "客户端幂等键" }
    },
    "required": ["priority", "title", "idempotency_key"]
  }
}
```

这样模型知道边界，OpenClaw 侧也能做参数校验。

### 3. 认证只放在 MCP server 层

API key、OAuth token 都放 MCP server 的环境变量或 secret 管理里。工具描述里不要出现任何密钥信息。MCP server 对外只暴露工具签名，不暴露认证细节。

### 4. 统一响应结构

MCP server 返回给 Agent 的内容最好归一化：

```json
{
  "ok": true,
  "data": { ... },
  "error_code": null,
  "request_id": "req_01H..."
}
```

出错时返回：

```json
{
  "ok": false,
  "data": null,
  "error_code": "rate_limited",
  "request_id": "req_01H...",
  "message": "上游限流，已自动重试，请稍后再试"
}
```

模型只读 `ok`、`error_code`、`message`，不读原始响应体。这样既省 token，也减少误判。

### 5. 超时、重试、错误映射放在握手层

不要在 Agent 逻辑里处理重试。MCP server 内统一设置：

- connect timeout 3s，read timeout 10s；
- 只对幂等的 GET/PUT 做自动重试；
- POST 类写操作必须带 `idempotency_key`，不自动重试，避免重复创建资源；
- 错误映射：`429/503 -> retryable`，`400/422 -> invalid_argument`，`401/403 -> auth_error`，其余 `upstream_error`。

### 6. 用 mock server 做测试

对接初期先用 mock server 返回固定响应，验证工具 schema、超时、错误映射是否符合预期。再切到真实 API，能少踩很多坑。

## 踩坑点

- **把 API key 写进提示词或工具描述**：很容易被 trace、日志、截图带走。密钥必须只存在于 MCP server 环境。
- **不设超时**：外部 API 卡住时，Agent 任务也会卡住。所有调用必须带超时。
- **把上游错误原样返回**：HTML 错误页或 Java 堆栈会严重污染模型上下文。必须映射成简洁的错误码和消息。
- **忽略分页**：查询列表接口如果不处理分页，模型可能只看到第一页就下结论。MCP server 内部要处理 `next_token` 或返回分页元信息。
- **没有 request_id**：上游排障时无法关联日志，只能靠时间猜。每次调用都生成或透传 request_id。
- **盲目重试非幂等操作**：创建订单、扣费、发通知这类接口不能自动重试。要么要求客户端传幂等键，要么明确不重试。

## 可复用建议

1. **一个系统一个 MCP server**，不要做通用网关。
2. **OpenClaw 侧配置工具白名单**，只让 Agent 看到必要的工具，降低误调用风险。
3. **写操作强制要求幂等键**，尤其是对接支付、工单、审批类系统。
4. **日志里记录 `tool_name`、`request_id`、`latency`、`status`**，方便排查是模型调用错还是上游挂了。
5. **先开放只读工具灰度**，跑稳定后再开放写操作。
6. **把错误码和重试策略写成文档**，让 Agent 能根据 `error_code` 做正确决策，而不是靠猜。

## 总结

OpenClaw 对接外部服务，核心不是“让模型能发 HTTP 请求”，而是把模型输出约束成安全的、可观测的 API 调用。MCP server 是合适的边界层：认证、超时、重试、错误归一化、分页都在这一层处理。OpenClaw 里的 Agent 只看到一个个明确的工具，不需要理解背后的 HTTP 细节。这样对接出来的外部服务，才真正可维护、可排障、可灰度。

---

