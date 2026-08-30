---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 35327
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

Agent 的价值不只是对话，而是能稳定地调用外部服务：查订单、建工单、发通知、读报表。在 OpenClaw 里，外部服务通常以 Tool、Plugin 或 MCP server 的形式暴露给 Agent。表面上是一次 HTTP 请求，实际上要解决参数生成、鉴权、超时、错误回传和结果裁剪等问题。

## 问题

如果让 Agent 直接裸调 HTTP，常见故障有三类：

- 参数不可控：日期格式、枚举值、必填项经常被模型自由发挥。
- 失败不可读：超时、429、500 全部变成一长段报错，Agent 容易反复硬试或直接中断。
- 边界不可控：返回体过大、副作用操作没有确认、密钥暴露在上下文里。

因此，对接外部服务时，应把 API 封装成边界清晰的工具，而不是让模型自己拼 URL 和 Header。

## 做法/步骤

### 1. 先拆工具，再写代码

不要做一个“万能 API 工具”。按资源或动作拆分，例如：

- `order_list`
- `order_cancel`
- `invoice_download`

每个工具只做一件事，副作用单独标注。这样模型选择更准，权限治理也更清晰。

### 2. 封装 API 客户端

把请求逻辑收到独立 client 层：

- base_url、token、timeout 放环境变量或 secrets；
- 连接超时建议 3～5 秒，总超时 15～30 秒；
- 只对 GET 或幂等请求自动重试；
- 统一返回结构，例如：

```json
{"ok": true, "data": {}, "error": null, "request_id": "req_xxx"}
```

`request_id` 对线上排障非常有帮助。

### 3. 注册到 OpenClaw

无论走 OpenClaw 插件还是 MCP server，核心都是写好工具定义：

- name 用 `动词_名词`；
- description 要说明“什么时候用、有什么副作用、返回什么”；
- parameters 明确类型、格式、枚举和默认值；
- 删除、取消、退款等高风险操作，应在 OpenClaw 侧开启人工确认或 require_confirmation。

一个简化的参数定义示例：

```json
{
  "order_id": {"type": "string", "description": "订单号，格式 ORD-xxxx"},
  "include_items": {"type": "boolean", "default": false}
}
```

这里的 description 是写给模型看的，要给出取值来源，而不是写“订单号”。

### 4. 错误映射

外部 API 的错误不要原样丢给 Agent。统一转换为：

- 可重试：超时、429、幂等的 5xx；
- 不可重试：400、401、403、404；
- 需要人工：余额不足、权限不足、数据冲突。

Agent 收到结构化错误后，才能决定换参数重试、放弃或请求人工。

### 5. 本地联调

先用 curl 或 mock server 验证外部 API，再用 OpenClaw CLI 单独调用新工具，最后才放进 Agent 流程。重点观察：

- 模型是否按 schema 填参数；
- 超时是否会卡住整个 run；
- 返回体是否过大。

## 踩坑点

1. **密钥写进 tool description**：模型可能把密钥当普通参数带出来。密钥必须留在 transport 层。
2. **返回体不裁剪**：外部 API 返回几 MB JSON，Agent context 直接爆掉。只保留必要字段，做分页或摘要。
3. **非幂等请求自动重试**：create、deduct、cancel 这类操作不要自动重试，容易重复扣款。
4. **副作用工具没有确认**：cancel、delete、refund 必须有人工确认或 gate。
5. **忽略 rate limit**：不读 `Retry-After` 或 429 的冷却时间，容易被外部服务封禁。

## 可复用建议

- 每个工具维护一张“工具卡”：场景、参数、返回示例、失败表现、rate limit。
- 统一返回 envelope，降低 Agent 解析成本。
- 鉴权放 client 层，工具层不关心 token。
- 外部 API 不稳定时，提供 fallback tool 或缓存策略。
- 监控 tool 的 P95 延迟、错误率、429 次数，比只看最终成功率更早发现问题。

## 总结

Agent 与 API 的握手，不是让模型学会发 HTTP 请求，而是把外部服务封装成模型真正能理解、能稳定使用、能安全退出的工具。清晰的 schema、可靠的错误映射、统一的返回结构和必要的人工确认，才是把外部服务纳入 Agent 工作流的关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/457c8565d7ecc7f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/113a43b03a3d947e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/63042b1bed83d348.png)

