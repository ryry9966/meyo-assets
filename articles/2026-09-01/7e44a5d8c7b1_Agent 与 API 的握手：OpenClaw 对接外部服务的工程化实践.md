---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 35590
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 里的 Agent 不是只靠对话完成任务，真正产生业务价值的地方在于它能否稳定调用外部 API：查订单、建工单、发通知、取报表。这一层“握手”做不好，Agent 再聪明也会变成只会聊天的 demo。

外部 API 和自然语言之间有天然落差：API 需要严格参数、鉴权、超时、错误处理，而模型倾向用自然语言“差不多”地填参数。对接外部服务本质上不是写一个 HTTP 请求，而是建立一个可执行的契约层。

## 问题

实际项目中，Agent 对接外部 API 的失败通常集中在几类：

- 模型乱填参数：字段缺失、类型错误、编造枚举值。
- 工具描述太模糊：Agent 不知道该在什么时机调用哪个工具。
- 密钥泄漏：把 API Key 写进工具描述或让模型直接接触。
- 超时和重试失控：外部服务抖动时，Agent 卡死或重复触发。
- 返回体不可解析：第三方 API 报错返回 HTML，Agent 解析 JSON 直接崩。

这些问题的共同点：没有把外部 API 的复杂性隔离在 Agent 之外。

## 做法

### 1. 用 JSON Schema 明确工具契约

在 OpenClaw 里定义一个工具时，不要把 endpoint 一填就完事。`description` 要写清楚“什么时候用、需要什么、返回什么、有什么边界”。参数用 JSON Schema 约束，不要用空对象或 `anyOf` 偷懒。

```yaml
tools:
  - name: order_query
    description: 查询最近 30 天订单。需要用户提供订单号或手机号，返回订单状态、金额和物流信息。
    parameters:
      type: object
      properties:
        order_id:
          type: string
          description: 订单号，例如 ORD20250101
        phone:
          type: string
          description: 用户手机号，order_id 和 phone 至少填一个
      required: []
    endpoint: http://127.0.0.1:8787/tools/order_query
```

### 2. 增加一层薄适配器

不建议让 OpenClaw 直接请求第三方 API。中间放一个轻量网关，哪怕只是 Node/Express 或 FastAPI 起的几十行服务。这层负责：

- 注入鉴权信息
- 参数补全与校验
- 统一返回结构
- 记录 request/response 日志
- 处理第三方 API 的非 JSON 错误

统一返回结构尤其重要。无论后端发生什么，给 Agent 的响应都保持同一种 shape：

```json
{
  "ok": true,
  "data": {},
  "error": null
}
```

这样 Agent 不需要理解各种第三方错误格式，判断 `ok` 字段即可。

### 3. 选择接入方式

OpenClaw 可以走内置工具配置，也可以接 MCP server。如果外部服务有状态、需要多步调用或复用性强，优先用 MCP 封装；如果只是一两个简单查询，内置工具加薄适配器足够。不要为了“上 MCP”而上 MCP，先看维护成本。

## 踩坑点

**不要在描述里暴露内部细节。** 例如写“调用 https://api.xxx.com/v1/order?key=sk-xxx”，模型可能把这段内容复述给用户。密钥始终由后端环境变量注入。

**不要给过宽的 schema。** 一旦出现 `additionalProperties: true` 或没有 `required`，模型会开始自由发挥。宁可先限制，后续需要再放开。

**不要忽略超时。** 工具请求必须设置合理超时，比如 8 秒。超时后给 Agent 返回可理解的错误信息，而不是让它无限等待。创建类接口要做幂等处理，防止模型超时后重试导致重复创建。

**不要把第三方返回直接透传。** 第三方返回 50KB 的 JSON，Agent 上下文很快被打爆。在适配层裁剪字段、分页、只保留任务需要的数据。

**不要只测正常路径。** 至少测三种情况：正常返回、业务错误、第三方超时/不可用。Agent 在异常路径下往往会重复调用或给出错误结论。

## 可复用建议

1. **每个工具一个契约文件**：把 input schema、output schema、错误码、超时时间写在一起，方便维护和评审。
2. **鉴权统一走服务端**：Agent 永远只拿一个内部 token，第三方密钥不进入模型上下文。
3. **适配层返回脱敏 trace_id**：出错时能快速定位到具体请求，同时避免把敏感信息喂给 Agent。
4. **能后端固定的参数不让模型填**：分页大小、排序方式、渠道来源等固定值直接写死在适配层，缩小模型的错误空间。
5. **给 Agent 留“失败出口”**：明确告诉工具，当 `ok: false` 时可以停止调用并向用户报告原因，不要无限重试。

## 总结

OpenClaw 对接外部服务的关键不是“能不能调通”，而是“模型能不能在合适的时候用对方式调通”。把 API 封装成有清晰契约的工具，外面再加一层隔离复杂度的适配层，能解决大部分工程问题。剩下的，就是把异常路径也当成一等公民来设计。

Agent 与 API 的握手，最终靠的不是 prompt 技巧，而是稳定的边界和可观测的失败。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/50bfe20f91aac53c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3e372d0efeff9873.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/cfd950b9460b4eeb.png)

