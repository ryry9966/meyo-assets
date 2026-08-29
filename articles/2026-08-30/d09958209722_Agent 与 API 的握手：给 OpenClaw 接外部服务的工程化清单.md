---
title: Agent 与 API 的握手：给 OpenClaw 接外部服务的工程化清单
feedId: 35295
source: 综合讨论
publishedAt: 2026-08-30
---

# Agent 与 API 的握手：给 OpenClaw 接外部服务的工程化清单

## 背景：Agent 不是万能客户端

给 OpenClaw 接外部服务时，很容易把 Agent 当成一个能自动理解 API 文档的万能客户端。实际上，Agent 对接口的理解只来自三样东西：工具描述、返回结构、报错信息。握手失败通常不是模型不够聪明，而是我们没把这三样喂清楚。

## 问题：直接调 API 会断在四个地方

1. 认证信息混在 prompt 或日志里；
2. 超时与重试没有边界，一个慢接口能卡住整条链；
3. 上游返回的非 200 或 HTML 错误页被当成 JSON 塞给模型，Agent 开始编造结果；
4. 写操作没有幂等，重试造成重复下单、重复建单。

这些问题不是 OpenClaw 独有，但它决定了你要不要在 Agent 侧再做一层收口。

## 做法：把 API 包成一个薄工具

我现在的习惯是：外部 API 不直接暴露给 Agent，而是在 OpenClaw 里注册一个单一职责工具，内部做三件事：校验入参、统一错误、控制重试。

示意配置（不是完整 schema，只表达边界）：

```yaml
tool:
  name: ticket_create
  description: Create a support ticket. Only use when user explicitly confirms.
  timeout_ms: 8000
  retry:
    max: 2
    backoff: exponential
  input_schema:
    title: string
    priority: low|normal|high
    content: string
  output:
    status: ok|error
    ticket_id: string|null
    error_code: string|null
    detail: string|null
```

工具内部：先校验入参，拼请求，设置 `Idempotency-Key`，带超时，非 2xx 时把上游 body 截断到 200 字符再放入 `detail`。然后让模型只读 `status/error_code/detail`，不接触原始响应。

在 OpenClaw 侧，我会把这类工具按域分组：查询类只读、写操作类要求用户确认、管理类需要额外审批。描述里写清“不会主动创建，除非用户明确说创建”。

## 踩坑点

- 把 API Key 写进工具描述，模型会在思考里复述出来；密钥要放环境变量或 secret store，只给工具运行时读取。
- 不设超时。默认超时可能过长，Agent 链会一直等；建议 5-10 秒上限，对慢接口用异步任务而不是同步等待。
- 把上游 HTML 错误页原样返回。模型看到一堆 script 标签会开始胡说；统一转成 `{ status: error, error_code, detail }`。
- 重试不区分方法。POST 没有幂等键时重试就是事故；只有 GET/HEAD 和带幂等键的 POST 可以自动重试。
- 输入 schema 太宽。`content: string` 不加长度上限，上游可能 413；在工具层限制长度，提前失败比让上游失败好。

## 可复用建议

1. 每个外部服务先写一个 mock server，用录制好的真实响应回放，快速验证 Agent 的行为；
2. 所有外部调用加 `request_id`，OpenClaw 侧日志、上游日志、返回错误都带同一个 id，排障不靠猜；
3. 工具描述用动词开头，写清楚“什么情况下不要用”，反向描述比正向更有用；
4. 把 Agent 的直接调用当成不可信客户端：限流、限权、限长度、限频率，不要因为它是你的 Agent 就开全量权限；
5. 用 MCP 接外部服务时，同样要注意 MCP server 的鉴权和超时，协议帮你解决传输，不解决业务错误。

## 总结

Agent 与 API 的握手，不是让模型读懂 API，而是你替它把 API 变成一个边界明确的函数：短超时、结构化错误、幂等写操作、密钥隔离。把脏活留在工具层，模型只做决策。这样 OpenClaw 接外部服务才会稳定，也不会在半夜重复建单。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/e53ac0baccdce63a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/89ea48b6b3f053f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9db27f6a65ff4a76.png)

