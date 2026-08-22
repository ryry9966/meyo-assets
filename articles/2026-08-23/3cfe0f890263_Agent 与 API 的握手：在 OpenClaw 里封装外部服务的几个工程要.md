---
title: Agent 与 API 的握手：在 OpenClaw 里封装外部服务的几个工程要点
feedId: 34290
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

OpenClaw 里，Agent 最常见的“对外动作”就是调用外部 API：查订单、建工单、取日志、发通知、操作 CRM。刚接第一个服务时通常很顺：写个 tool，塞进 prompt，LLM 自动会调。但接第三个、第五个服务后，问题开始出现——超时卡死、重复写操作、错误信息不可读、上下文被大 JSON 撑爆。

这本质上不是 Agent 不够强，而是“握手层”没做好。OpenClaw 负责编排与工具调用，但外部服务的可靠性、错误语义、幂等边界，需要我们自己封装。

## 问题

直接让 LLM 调裸 API，通常会踩这些坑：

- 超时设置缺失或过短，Agent 卡在工具调用上
- 上游返回 500 / 429，模型看到原始 HTML 或堆栈，开始瞎猜
- 创建类接口没有幂等键，重试导致重复数据
- 分页接口一次返回几万行，上下文直接爆掉
- 鉴权信息散落在描述文本或参数里，容易泄漏
- 工具定义过于宽泛，LLM 自由发挥参数，错误率升高

## 做法 / 步骤

我建议每个外部服务都走一个“薄适配层”，而不是让 OpenClaw 直接面对第三方 API。

### 1. 先定义工具契约

不要直接把 API 文档丢进 description。给每个工具定义清晰的 JSON Schema，限制枚举、默认值、必填项。示例：

```yaml
name: create_support_ticket
description: Create a support ticket. Returns ticket_id on success.
parameters:
  subject:
    type: string
    maxLength: 120
  priority:
    type: string
    enum: [low, normal, high]
    default: normal
  idempotency_key:
    type: string
    description: Unique key for safe retry
required: [subject]
```

### 2. 封装 HTTP 客户端

统一使用一个 `call_external` 封装，而不是在 tool 里随意写 `requests.post`。要点：

- 设置 `timeout=(connect_timeout, read_timeout)`
- 只对 GET 或明确幂等的请求自动重试
- 401、403、429、5xx 分别映射为结构化错误
- 所有请求自动带 `User-Agent`、trace id、超时时间

### 3. 统一返回 envelope

外部 API 返回的数据不要直接给模型。先包一层：

```json
{
  "ok": true,
  "data": {"ticket_id": "T-1042"},
  "error": null
}
```

失败时：

```json
{
  "ok": false,
  "data": null,
  "error": {
    "code": "UPSTREAM_TIMEOUT",
    "message": "Support service did not respond within 15s",
    "retryable": true,
    "retry_after_ms": 2000
  }
}
```

这个 envelope 对 LLM 非常友好：模型能根据 `retryable` 和 `retry_after_ms` 决定下一步，而不是读原始异常。

### 4. 在 OpenClaw 侧注册

如果 OpenClaw 支持 MCP，可以把适配层做成 MCP server；如果走内置工具注册，就暴露同结构 handler。核心是：**OpenClaw 看到的永远是稳定、带 schema 的工具，而不是裸第三方 API。**

### 5. 先用 cURL 和 mock server 验证

不要一接好就交给 Agent 自由跑。先用 cURL 验证真实接口，再用本地 mock server 覆盖超时、429、500、大响应等场景。这样可以验证错误归一化和重试逻辑。

## 踩坑点

### 1. 重试导致重复写操作

最常见也最危险。解决方式：所有创建/更新类工具必须包含 `idempotency_key` 或由适配层生成 `request_id`，并传给上游。不要依赖“模型不会重试”的假设。

### 2. 超时设置不考虑服务 p95

有些第三方服务正常响应就要 8–12 秒，如果 `timeout=3s`，会大量误报；如果设 60s，OpenClaw 工具调用会长时间挂起。按服务 p95 和用户可接受的等待时间设置，例如 connect 2s、read 20s。

### 3. 分页和大响应

外部接口一页返回 5000 条，OpenClaw 上下文直接不可用。适配层需要做截断、摘要或游标分页，只返回前 N 条或按条件过滤后的结果。

### 4. 鉴权信息泄漏

不要在 tool description 或参数 schema 里写 token。使用环境变量注入，日志中脱敏。本地开发时如果服务跑在 localhost，注意容器/沙箱环境网络隔离，可能需要 `host.docker.internal` 或配置内网域名。

### 5. 错误消息不可行动

原始错误如 `Connection reset by peer` 对 LLM 没有意义。错误归一化时尽量给出可行动建议：`retry_after_ms`、`check API key`、`reduce page_size` 等。

## 可复用建议

- **适配层与 Agent 解耦**：一个外部服务一个适配文件，内部包含 schema、client、错误映射，OpenClaw 只负责调用。
- **返回永远用 envelope**：让 LLM 只处理 `ok/data/error` 三种情况。
- **写操作必须幂等**：即使上游不支持，也可以在适配层生成 client id 作为数据库唯一约束。
- **记录每次外部调用日志**：请求耗时、状态码、错误码、是否重试，方便回溯 Agent 行为。
- **工具数量保持克制**：不要让一个 Agent 同时面对 20 个外部服务。可以按场景分组，减少选择压力。
- **用 mock 记录回放**：将真实响应录下来，作为离线测试用例，避免频繁打三方接口。

## 总结

OpenClaw 对接外部服务，难点不在“让 Agent 会调 API”，而在握手层的工程可靠性。把超时、鉴权、错误映射、幂等、上下文预算处理好之后，Agent 才能真正稳定地干活。否则，每一次第三方抖动都会变成 Agent 的随机失败。

换句话说：**Agent 负责决策，适配层负责握手；握手不稳，决策再好也执行不下去。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/5648e6366e3a66f1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/955efed875ab89c4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ca24bce433f980b3.png)

