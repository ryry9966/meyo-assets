---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 35610
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景：Agent 的脆弱点不在模型，在外部依赖

OpenClaw 的 Agent 循环通常依赖 MCP 工具或插件调用外部 API：搜索、天气、数据库、支付、文件存储等。模型本身对“工具失败”没有工程直觉。如果 API 超时、返回 500 或限流，Agent 可能重复调用同一个工具，或者在错误堆栈里“编”一个结果。更危险的是，某些失败发生在多步任务中间：前一步已写入数据，后一步失败，留下半完成状态。错误恢复不能只靠 prompt 里写“请重试”，需要在工具接入层做工程化处理。

## 问题拆解

- **重试风暴**：没有退避和上限的重试，短时间打满上游。
- **错误信息污染上下文**：原始异常堆栈很长，LLM 容易抓错重点，产生幻觉。
- **无降级路径**：API 挂掉后，Agent 没有替代数据源，任务直接中断。
- **副作用不可控**：非幂等写操作被重试，造成重复扣款/重复创建。
- **熔断缺失**：上游已经不可用，Agent 仍持续尝试，浪费时间和 token。

## 做法与步骤

### 1. 在工具层统一错误处理

不要直接把异常抛给 Agent。在 OpenClaw 插件或 MCP server 的出口包一层 `withApiRecovery`。核心逻辑：

- 将错误分类为 `retryable`（超时、429、5xx）、`non-retryable`（400、401、403）、`degradable`（有缓存可降级）。
- 对 retryable 使用指数退避 + 随机抖动，例如 `base * 2^attempt + random(0, 100ms)`，最多重试 2-3 次。
- 设置硬超时（如 8s），避免 Agent 长时间等待。

### 2. 加入熔断器

对每个外部 API 维护一个熔断状态。连续失败 3 次后打开熔断，冷却 30s；冷却后进入半开状态，只允许一个探测请求。如果成功则关闭，失败则重新打开。这个状态可以放在内存或 Redis 中，与 Agent 进程解耦。

这样做的好处是：当上游已经整体不可用时，Agent 后续调用会快速失败，并立即走降级路径，而不是继续占用任务时间。

### 3. 设计降级策略

每个关键 API 都应准备降级方案。例如：

- **搜索 API 挂掉**：返回最近 24h 的本地缓存结果，并标注 `stale=true`。
- **天气 API 超时**：返回默认季节数据，或直接告知 Agent“天气数据不可用，跳过该步骤”。
- **支付 API 失败**：绝不自动降级，转入人工确认队列。

降级结果必须带上明确的元数据，让模型知道这是降级数据，而不是真实成功。例如返回 `{ "source": "fallback", "freshness": "12h" }`。

### 4. 保证写操作幂等

重试之前必须考虑副作用。对创建订单、发送消息、写入数据库等操作，使用幂等键。OpenClaw 插件可以在调用前生成 `idempotency_key`，重试时复用同一个 key。若下游不支持幂等，则对于写操作最多重试 1 次，且失败后立即标记为 `needs_human_review`。

### 5. 向 Agent 返回结构化错误摘要

Agent 不需要看到完整堆栈。将工具错误转换为简短、可操作的信息：

```json
{
  "status": "failed",
  "error_type": "timeout",
  "suggestion": "retry after 2s or use cache",
  "fallback_available": true
}
```

这能显著降低 LLM 的幻觉概率，并帮助它做出合理决策。

## 踩坑点

- **对所有错误都重试**：400 参数错误重试 3 次毫无意义，只会消耗配额。先分类再重试。
- **超时设置过长**：OpenClaw 的任务超时通常有限，外部 API 超时设 15s 以上会拖垮整个 Agent 循环。
- **熔断状态在 Agent 重启后丢失**：如果熔断器在进程内，重启后全部重置，可能立即再次打爆上游。建议持久化到 Redis。
- **降级数据过期导致错误决策**：缓存超过一定时间后应视为不可用，不能无限降级。
- **忽略 Retry-After 头**：遇到 429 应该读服务端给的等待时间，而不是自己猜。
- **半完成写入未清理**：多步任务中间失败，要提供补偿或回滚逻辑，否则数据不一致。

## 可复用建议

在 OpenClaw 中，可以统一封装一个 `reliableTool` 工厂函数：

```
const searchTool = reliableTool({
  name: "search",
  executor: api.search,
  retry: { maxAttempts: 3, backoff: "exponential" },
  timeout: 8000,
  circuitBreaker: { failureThreshold: 3, cooldown: 30000 },
  fallback: cacheSearch,
  idempotent: false
});
```

所有 MCP 插件都通过类似方式接入。同时建议在配置文件中启用 `logToolFailures`，记录每次失败的错误类型、重试次数、是否降级，便于事后排查。

测试时可以注入故障：让 API 返回 500、超时、429，观察 Agent 是否按预期降级或暂停，而不是陷入循环。

## 总结

外部 API 故障是 Agent 工程中的常态，而不是异常。把错误恢复做在工具接入层，而不是全靠模型临场发挥，可以大幅提升任务完成率和可观测性。核心原则是：**分类错误、有限重试、快速失败、明确降级、保持幂等、留下痕迹**。这套逻辑不复杂，但需要在每个外部依赖上持续执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/582f8188130995f8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/358c073f8268b2d9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d7ba91e46f18075d.png)

