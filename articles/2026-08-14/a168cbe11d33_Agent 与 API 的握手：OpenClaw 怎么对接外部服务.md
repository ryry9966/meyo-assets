---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 33089
source: 综合讨论
publishedAt: 2026-08-14
---

# Agent 与 API 的握手：OpenClaw 怎么对接外部服务

## 背景

把 Agent 接上外部 API 并不难，难的是接得稳定、可维护、不把模型带偏。很多自动化场景里，OpenClaw 需要查询订单、创建工单、读取监控指标、触发部署。外部服务通常是给前端或后端系统用的，不是给 LLM 用的：字段多、返回大、错误码不统一、限流严格。

如果直接让模型发 HTTP 请求，或者把原始 API 返回整个丢给模型，会出现三个典型问题：上下文被无意义字段撑爆、凭证管理混乱、错误信息不可读导致模型反复重试。

所以 OpenClaw 对接外部服务，本质上不是“能调通”，而是要把外部 API 包成一个模型友好的工具。

## 问题

常见外部 API 有两个特征：

1. **返回结构冗余**。一个订单查询可能返回 40 个字段，但 Agent 只需要 `status`、`tracking_no`、`eta`。
2. **错误语义不明**。返回 401、429、5xx 时，Body 可能是 HTML、网关错误页或堆栈信息，模型看了也不会处理。

如果直接把原始响应喂给模型，工具调用会变得不可控。更糟的是，模型可能把敏感字段带到下一步输出里。

## 做法：用薄封装代替裸调

OpenClaw 里推荐的做法是声明式工具配置，只暴露必要能力。下面以常见的 HTTP 工具配置为例，不同版本字段可能有差异，以你部署的 schema 为准。

### 1. 声明工具和鉴权

```yaml
tools:
  - name: order_lookup
    description: 根据 order_id 查询订单状态，返回 status、tracking_no、eta
    endpoint:
      url: ${ORDER_SERVICE_BASE_URL}/v1/orders/{order_id}
      method: GET
      headers:
        Authorization: Bearer ${ORDER_SERVICE_TOKEN}
    parameters:
      order_id:
        type: string
        required: true
        description: 订单号，只接受数字，不能传邮箱或手机号
    timeout_ms: 5000
    retry:
      max_attempts: 2
      backoff: exponential
```

凭证通过环境变量注入，不要写死在配置或前端代码里。

### 2. 裁剪返回字段

不要让模型看到完整 JSON。用 `response_mapping` 只取需要的字段：

```yaml
    response_mapping:
      status: data.attributes.status
      tracking_no: data.attributes.tracking_number
      eta: data.attributes.estimated_delivery
```

这样模型拿到的是一个小对象，而不是几十个嵌套字段。上下文占用更小，工具输出也更稳定。

### 3. 错误归一化

把 HTTP 错误码映射成模型能理解的中文短句：

```yaml
    error_mapping:
      401: "认证失败，请检查 ORDER_SERVICE_TOKEN"
      404: "未找到该订单"
      429: "订单服务限流，请稍后重试"
      5xx: "订单服务暂不可用，不要立即重试"
```

这一步很重要。模型看到“订单服务暂不可用”会停止重试；看到一坨 HTML 则可能开始瞎猜。

### 4. 只读接口加缓存

查询类工具可以加 30–60 秒缓存，避免模型连续追问时打爆外部 API。写操作不要缓存，并且要避免无脑重试。

## 踩坑点

- **直接暴露原始 API**：字段爆炸、敏感信息泄漏、错误不可读。薄封装是底线。
- **重试不区分幂等**：GET、PUT 可以重试；POST 触发类接口要谨慎，可能重复创建资源。
- **工具描述太模糊**：模型传错参数比 API 挂掉更常见。描述里要写清楚参数格式和限制。
- **schema 变更静默失效**：外部 API 升级后，`response_mapping` 可能取不到值，返回空对象。需要监控工具输出为空的比例。
- **限流被模型放大**：Agent 会在一次任务里连续调用工具，外部 API 很容易被 429 打挂。加缓存和 rate limit 是必要的。

## 可复用建议

1. **做一套外部服务接入模板**：统一处理鉴权、超时、重试、错误归一、字段裁剪。每个新服务只改 endpoint 和 mapping。
2. **工具描述按四段写**：做什么、输入参数、返回内容、限制条件。
3. **读多写少**。查询接口可以宽松缓存，写接口要单独标记幂等性。
4. **用 trace 看真实请求**。不要只看模型回答，OpenClaw 的 trace 或 dry-run 能看到实际请求体和响应，排查参数传错非常有效。
5. **凭证只走环境变量**。配置文件中不要出现真实 token，发布前做脱敏检查。

## 总结

OpenClaw 对接外部服务，核心不是“接上”，而是“接稳”。把外部 API 包成模型友好的薄工具，做好字段裁剪、错误归一、重试策略和限流保护，才能让 Agent 与 API 的握手真正可靠。工程上多花一点时间在封装层，后面自动化跑起来会省很多排障成本。

---

