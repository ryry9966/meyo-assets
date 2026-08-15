---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 33310
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

OpenClaw 作为 Agent 运行时，本身不产生业务数据，真正有价值的部分是它能按需调用外部 HTTP API、内部 RPC 或 MCP 服务。对接外部服务不是“填一个 URL”就能完成，中间隔着鉴权、超时、错误语义、限流和可观测性。工程上做得粗糙，Agent 会频繁误判、卡死或泄露密钥。

## 问题

实际项目中常遇到三类问题：

- **接口散落**：每个开发者写的 tool handler 风格不一，有的忽略超时，有的把密钥写进代码。
- **错误难懂**：外部 API 返回 400/429/500 时，Agent 拿到的是堆栈或原始 HTML，容易误判为工具本身故障。
- **不可复用**：同一个 API 对接逻辑在不同 tool 里重复实现，改一个鉴权要动多处。

## 做法/步骤

以一个最小但完整的场景为例：让 OpenClaw 调用一个需要 Bearer Token 的订单查询 API。

### 1. 先定义 tool schema

OpenClaw 的工具注册一般需要 `name`、`description`、`input_schema`。描述要写明“什么时候用”，而不只是“查订单”。例如：

```yaml
name: query_order
description: 根据订单号查询订单状态。当用户询问某个订单是否发货、支付状态时使用。
input_schema:
  type: object
  properties:
    order_id:
      type: string
      description: 订单号，形如 ORD-20240101-123
  required: [order_id]
```

`description` 写清触发场景，能显著减少 Agent 误调用。

### 2. 实现 handler，统一 HTTP 行为

不要在 tool 内部直接裸调 HTTP 客户端。建议先封装一个 `BaseAPITool` 或统一 HTTP client，处理：

- **超时**：连接 2s，读取 8s，总超时 10s。外部服务慢时宁可失败重试，也不要让 Agent 卡住。
- **请求头**：`Authorization` 从环境变量读取，不写死在代码。
- **错误映射**：把非 2xx 转成 Agent 友好的短错误，例如“订单服务返回 429，请稍后重试”，不要返回原始 HTML。

### 3. 错误与重试

只对幂等 GET 做自动重试，最多 2 次，间隔采用指数退避加抖动。POST 请求默认不自动重试，除非业务明确支持幂等键。遇到 429 时读取 `Retry-After` 头，按服务端建议等待，而不是盲目重试。

### 4. 接入 OpenClaw 工具注册

将上述 handler 注册为 OpenClaw 工具，通常在插件或工具目录里新增文件。确认工具能被 Agent 发现后，先在 CLI 或调试面板手动触发一次，再让模型调用。

### 5. 配置注入

密钥、Base URL、环境隔离通过环境变量或 OpenClaw 的配置管理注入。不要在 schema 里暴露 token，也不要在日志里打印请求头。

## 踩坑点

- **Schema 过宽或过窄**：过宽会让模型生成无关字段，过窄会导致正常参数传不进来。建议逐步收紧，别一上来就 `anyOf` 或空 `properties`。
- **超时设置不匹配**：某些内部 API P95 超过 8s，但 Agent 默认 5s 超时，频繁失败。先看服务端延迟分位数再设超时。
- **日志泄露**：打印完整请求 URL 时，可能带 query 参数里的 token；打印响应时可能带用户敏感信息。日志必须脱敏。
- **限流误判**：429 不是“服务挂了”，不应该无限重试，也不该直接告诉用户失败。重试耗尽后要给出明确建议。
- **分页处理**：外部 API 返回列表时，不要在 tool 里一次性拉全量，容易超时。先返回第一页和总数，让 Agent 决定是否继续翻页。

## 可复用建议

- **封装为 MCP server**：把外部 API 封装成 MCP server，而不是每个 tool 直接调 HTTP。MCP server 可以独立部署、限流、鉴权，OpenClaw 只关心工具契约，多个 Agent 或工具可复用同一 server。
- **做 BaseAPITool**：内部包含重试、超时、结构化错误、审计日志。新接口只需实现 `parse_response` 和 `build_request`。
- **指标与健康检查**：给每个外部依赖建调用量、失败率、P95 延迟、限流次数。外部服务抖动时能快速定位。
- **保留 curl 脚本**：用于脱离 Agent 排查。不要等 Agent 表现异常才去查 API，先用 curl 验证接口本身是否正常。

## 总结

OpenClaw 对接外部服务的难点不在 HTTP 请求本身，而在于把不稳定的外部世界翻译成 Agent 能理解、可控的稳定接口。工程化做法是：清晰 schema、统一 base client、安全配置、明确错误语义、可观测指标。做到这些，Agent 调用外部 API 才能从“能跑”变成“可维护”。

---

