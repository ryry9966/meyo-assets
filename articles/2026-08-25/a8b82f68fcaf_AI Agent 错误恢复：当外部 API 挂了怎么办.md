---
title: AI Agent 错误恢复：当外部 API 挂了怎么办
feedId: 34704
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里，Agent 经常通过 MCP 工具或插件访问外部 API：搜索、支付、CRM、地图等。外部 API 不可避免会挂：超时、限流、网关 502、上游抖动。问题在于 Agent 是“自主决策”的。如果工具把原始异常抛给模型，模型可能反复调用、误判错误、编造结果，或者直接中断流程。错误恢复不能只靠提示词，必须在工具层和策略层兜底。

## 问题

典型失败链：API 5xx -> 工具返回异常 -> 模型看到报错 -> 再试一次 -> 仍失败 -> 上下文堆积错误 -> 用户等待超时或得到错误结论。更糟的是，模型把 401 当可重试，反复触发风控。需要明确：哪些错误值得重试、怎么退避、如何熔断、失败后如何降级、如何让 Agent 如实传达。

## 做法/步骤

### 1. 工具层统一错误包装

不要让 MCP 工具直接返回原始 traceback。给外部 API 调用加一个 wrapper，把错误转成结构化 JSON：

```json
{
  "ok": false,
  "error_type": "upstream_timeout",
  "retryable": true,
  "retry_after": 2,
  "fallback_used": false,
  "hint": "Upstream timed out. Retry once or use cached data if available.",
  "detail": "sanitized short message"
}
```

错误类型分三类：可重试（网络超时、429、5xx）、不可重试（400、401、403、404、业务错误）、可降级（有 fallback 可用）。这个结构对模型友好，也能防止敏感信息进入上下文。

### 2. 重试策略：退避 + 幂等边界

对可重试错误启用指数退避和 jitter，避免大量 Agent 同时重试造成雪崩。例如 base 200ms、factor 2、最多 3 次，总耗时控制在 5 秒内。尊重 429 的 Retry-After。

关键边界：只对幂等操作自动重试。GET/查询类可以重试；POST/PATCH 如果没有幂等键，不要盲重试，否则可能重复创建订单、重复扣款。工具描述里应标记是否幂等，由 wrapper 判断。

### 3. 超时与熔断

每次外部调用都要设置 connect timeout 和 read timeout。OpenClaw 插件/工具不能无限等，否则 Agent 长时间挂起。连续失败 N 次（如 3 次）后打开熔断，快速失败 30 秒，后续请求直接返回“服务暂不可用”，避免继续打已经过载的 API。熔断状态要可观测，暴露到日志/指标里。

### 4. 降级与缓存

给关键只读数据准备 fallback：本地缓存、只读副本、第二供应商。命中降级时，返回体要带 `"fallback": true` 和 `"stale": true`，并告诉模型数据可能不是最新。禁止静默降级，否则 Agent 会拿着过期数据做决策。

### 5. 上下文保护与断点恢复

控制错误信息长度：不要塞完整响应体、堆栈、密钥。给模型的是 sanitized 摘要。多步任务中途失败时，保留已完成步骤的状态；支持从失败步骤恢复，而不是从头再来。

### 6. 提示词只给策略，不负责执行

系统提示可以写明：如果工具返回 retryable 且已重试失败，就直接向用户报告，不要自己另想奇怪办法；如果返回 fallback/stale，必须说明数据可能过期。但重试、熔断逻辑不要全写进提示词，模型可能不遵守或过度发挥。

## 踩坑点

- 无限重试打爆上游：尤其 5xx 时，上游可能已经过载，重试加重故障。
- 对 401/400 重试：不仅没用，还可能触发风控或锁定账号。
- 非幂等写操作自动重试导致重复业务数据。
- 没有超时，Agent 卡死，用户以为程序在跑。
- 上下文污染：完整堆栈让模型抓不住重点，甚至把错误码当数据。
- 静默降级：用旧数据但不标记，导致后续动作错误。
- 只在提示词里要求“遇到错误要优雅处理”，没有工具层兜底。

## 可复用建议

- 做一个统一的 tool wrapper：处理错误分类、重试、超时、熔断、降级，MCP/插件可复用。
- 配置化：retry_count、timeout_ms、breaker_threshold、cache_ttl 按 API 配置。
- 错误协议化：所有工具返回相同 JSON schema。
- 重要 API 准备降级路径：缓存、备用供应商、人工接管。
- 定期做故障演练：手动 kill 依赖、mock 429/502，看 Agent 是否能在一分钟内停止重试并如实汇报。
- 观测指标：失败率、重试次数、熔断打开次数、降级命中率、P95 延迟。

## 总结

外部 API 挂了不是偶发，而是工程常态。AI Agent 的错误恢复核心不是让模型更聪明，而是把不确定性封装在工具层：结构化错误、有限重试、熔断、降级、上下文保护。这样 Agent 才能从“报错卡死”变成“快速失败、如实汇报、最小化影响”。对 OpenClaw 实践而言，一个可靠的 wrapper 比一段更长的系统提示词有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/41c232aeb7c305cd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/99a1d511fa3212c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/62fc76f433e92b42.png)

