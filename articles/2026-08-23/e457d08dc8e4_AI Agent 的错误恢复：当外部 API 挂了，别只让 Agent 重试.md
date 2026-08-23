---
title: AI Agent 的错误恢复：当外部 API 挂了，别只让 Agent 重试
feedId: 34320
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的工具、MCP server 和插件链路里，外部 API 是常见依赖：搜索、天气、支付、对象存储、LLM 供应商等。依赖会超时、限流、返回 5xx，甚至直接断连。Agent 任务链往往会因此中断。

## 问题

很多实现的处理方式很原始：try/except 加固定重试，或者把异常原样丢给模型。结果不是恢复，而是放大故障：重试风暴占满配额，非幂等操作被重复执行，模型看到原始堆栈后开始“编造”原因，错误被掩盖到最终输出里。

## 做法/步骤

1. **在工具层定义统一错误契约**  
   给每个工具返回结构化结果，而不是裸异常。至少包含：`error_code`、`retryable`、`retry_after`、`fallback_hint`、`source`。错误先分类：`INVALID_REQUEST`（4xx 参数/配额/权限，不重试）、`RATE_LIMITED`（429，退避）、`UPSTREAM_UNAVAILABLE`（5xx/超时，可重试）、`TIMEOUT`。

2. **重试只对可重试错误生效**  
   指数退避加抖动，限制最大次数和总时长；429 优先读 `Retry-After`；非幂等工具默认不自动重试。

3. **加熔断器**  
   同一工具连续失败达到阈值后快速失败，冷却期后半开试探，避免上游被持续打挂。

4. **做降级**  
   只读工具优先用缓存兜底；无缓存时返回空结果，并标记 `source=fallback`；关键链路可配置备用 API。

5. **把健康状态回传 Agent**  
   熔断、降级、不可用信息进入工具返回或系统提示，让 Agent 知道当前哪些能力缺失，能主动换工具或调整计划。

6. **补可观测性**  
   记录 `error_code`、`retry_count`、`duration_ms`、`circuit_state`，错误摘要回传而不是原始堆栈。

一个简化的包装示例：

```ts
const res = await withRecovery(() => api.search(q), {
  retry: { max: 3, backoff: [1, 2, 4], jitter: true, onlyIdempotent: true },
  circuit: { threshold: 5, cooldownMs: 30_000 },
  fallback: () => cache.get(q) ?? { ok: true, source: 'fallback', items: [] }
});
```

## 踩坑点

- 所有异常都重试：4xx、配额、权限错误会越试越糟。
- 对 POST/支付/创建类工具自动重试，可能导致重复扣款或重复创建。
- 把原始堆栈贴进 LLM 上下文，模型会按异常文本乱推。
- 降级结果不标 `source`，Agent 把缓存数据当实时结果。
- 只处理工具错误，不管 MCP server 断连或插件超时，任务悬挂。
- 熔断后没有告警或人工入口，问题一直被吞。

## 可复用建议

- 在 OpenClaw 工具/MCP wrapper 统一实现 `ToolResult` 和恢复策略，不要散落各工具里。
- 默认策略：只读工具可重试 3 次并降级；写操作返回可重试标记但不自动重试。
- MCP server 增加健康检查、超时与取消，单个慢工具不拖垮整个任务。
- 用故障注入测试：手动让上游返回 500/429/断连，观察 Agent 是否按预期降级、跳过或告警。
- 给模型的错误信息保持一行：`search_web failed: RATE_LIMITED, retry_after=2s, source=fallback`。

## 总结

外部 API 挂掉时，Agent 的恢复能力不取决于“重试多少次”，而取决于错误分类、退避、熔断、降级和可观测性是否下沉到工具层。目标是让 Agent 在依赖不可用时仍然可控，而不是假装一切正常。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1a7b5c9384f57fa8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8f2aeaad9f872584.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/45e004117b0c5a7a.png)

