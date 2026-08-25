---
title: 外部 API 挂了，Agent 别跟着挂：一套可落地的错误恢复设计
feedId: 34752
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw 或任何 Agent 编排里，外部 API 不是“偶尔会挂”，而是长期运行一定会遇到的状态。天气、搜索、支付、OCR、模型供应商、向量库，都可能超时、限流、返回 5xx。

Agent 和普通脚本最大的区别是：脚本失败会直接退出，Agent 失败后模型会“想办法”。这个“想办法”如果不受控制，就会变成重复调用、幻觉修复、上下文膨胀，最后任务失败且账单很高。

## 问题

常见的错误处理模式通常有这些问题：

- 工具把异常原样丢给模型，模型开始读堆栈，然后尝试“修复”。
- 不区分 4xx 和 5xx，所有错误都无脑重试。
- 没有熔断，上游已经过载时 Agent 还在持续打。
- 没有降级，只读数据拉不到就整个任务断掉。
- 没有幂等，写操作重试造成重复扣款、重复发消息。

外部依赖不可用应该被当成常态，而不是意外。

## 做法 / 步骤

### 1. 让工具返回结构化错误

MCP 工具或插件函数不要只抛 exception。统一的错误对象更可控：

```json
{
  "ok": false,
  "retryable": true,
  "code": "UPSTREAM_TIMEOUT",
  "message": "upstream timed out after 3s",
  "retryAfterMs": 1200,
  "fallback": null,
  "stale": false
}
```

字段含义固定：`ok` 表示是否成功，`retryable` 表示是否值得重试，`code` 是稳定错误码，`fallback` 是降级数据，`stale` 表示降级数据是否过期。这样模型决策时只需要看这几个字段，不需要读堆栈。

### 2. 设置退避与重试上限

在工具内部做重试，而不是让模型自己反复调用。只读请求最多重试 2-3 次，指数退避加抖动；写请求默认不自动重试，除非已经做了幂等。

```text
max_retries = 2
base_delay = 500ms
for attempt in 1..max_retries:
    resp = call_api(timeout=3s)
    if resp.ok: return resp
    if not resp.retryable: break
    sleep(base_delay * 2^(attempt-1) + random_jitter)
return fallback_cache or fatal_error
```

### 3. 加熔断

连续错误超过阈值时，短路 30-60 秒。熔断期间工具直接返回 `{ok:false, retryable:false, code:"CIRCUIT_OPEN"}`，或者走本地缓存。避免上游已经过载，Agent 还继续往里打。

### 4. 缓存兜底

只读 API 适合做 TTL 缓存。失败时返回 last known good，并标记 `stale:true`。模型在提示词里需要知道：stale 数据只能用于非关键判断，必要时告知用户“数据来自缓存”。

### 5. 用提示词约束模型行为

在 Agent 系统提示中加几条：

- 工具返回 `retryable=true` 且重试次数未超限，再调用一次；
- `retryable=false` 不要继续硬试；
- 不要根据错误堆栈猜测原因；
- 相同参数不要重复调用超过 N 次；
- 如果工具返回 `CIRCUIT_OPEN`，切换备选方案或暂停后询问用户。

### 6. 持久化状态与幂等

长任务要 checkpoint。失败恢复时，从最后一个成功步骤继续，不重复执行支付、发送消息等副作用步骤。写操作必须带幂等键。

## 踩坑点

- **把 400/401/403 当可重试**：权限和参数错误重试多少次都不会变好。
- **无限重试拉高 token 和费用**：模型反复调用同一工具，每一轮都消耗上下文和 API 费用。
- **把完整堆栈抛给模型**：堆栈会污染上下文，模型容易“对着堆栈幻想修复方案”，越修越偏。
- **写接口没有幂等**：重试造成重复扣款、重复下单或重复发消息。
- **降级数据不标记 stale**：模型把 30 分钟前的缓存当实时数据，做出错误决策。
- **没有总任务 deadline**：单个步骤都有超时，但整体任务可能因为多个步骤串行重试拖到 20 分钟。

## 可复用建议

1. 为所有 MCP 工具/插件统一错误协议，至少包含 `ok`、`retryable`、`code`。
2. 读接口优先做缓存兜底，写接口优先做幂等，再考虑重试。
3. 把重试次数、熔断阈值、超时时间做成配置，而不是硬编码。
4. 给 Agent 设置“步骤重试上限”和“任务总时长预算”，防止雪崩。
5. 错误日志记录结构化元数据：调用参数、耗时、状态码、是否降级，方便事后评估。
6. 关键外部依赖准备一个备选工具或降级路径，比如主搜索挂时用备用搜索，或返回历史结果并提示用户。

## 总结

外部 API 挂掉不可怕，可怕的是 Agent 把它当成一个需要“不断尝试解决”的问题。工程上更合理的目标不是保证 100% 成功，而是：快速失败、有界重试、熔断保护、降级兜底、任务可恢复。把错误处理做在工具层，把决策边界交给 Agent，而不是让模型自己发挥。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/ea07a0126c564f73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/01ea0f5073a803e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/2fcd0aa454ea5b38.png)

