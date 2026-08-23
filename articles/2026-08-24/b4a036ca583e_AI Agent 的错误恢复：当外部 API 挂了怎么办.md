---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 34403
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw 这类 Agent 自动化流程里，外部 API 不稳定是常态，不是意外。LLM 供应商限流、搜索接口 502、天气服务超时、MCP server 进程崩溃，都可能让一条原本能跑通的任务链中途断掉。更麻烦的是，很多 Agent 工具调用默认让异常直接冒泡，导致整个 run 失败；如果用户无脑重跑，前面已经执行过的副作用步骤还会重复发生。

## 问题

错误恢复不是简单加 `try/except`，而是需要回答四个问题：

1. 这次错误能不能重试？
2. 重试多久、退避多快？
3. 失败后 Agent 应该继续、跳过，还是停止？
4. 哪些状态已经安全落盘，哪些步骤可以降级？

如果这些问题没有明确策略，Agent 要么在单个接口上卡死，要么在重试中重复扣费/重复发送。

## 做法 / 步骤

### 1. 先做错误分类

把外部错误分成两类：

- **可重试**：网络超时、429、502、503、504
- **不可重试**：401、403、422、业务拒绝、参数错误

不要对不可重试错误做重试，那只会放大问题。

### 2. 统一调用包装器

在 OpenClaw 的工具层加一个 `external_call` 装饰器或中间件，统一处理超时、退避和降级。示例伪代码：

```python
@external_call(
    max_retries=3,
    base_delay=0.5,
    max_delay=5,
    jitter=True,
    timeout=10,
    retry_on=[TimeoutError, 429, 502, 503, 504],
    fallback="cache_or_default"
)
def fetch_search_results(query: str):
    ...
```

这里的关键是：**重试逻辑不散落在业务代码里**，否则每个工具实现都会长得不一样。

### 3. 设置超时预算

单个工具要有 timeout，整个 Agent run 也要有 deadline。指数退避如果设置得太长，可能重试还没结束，上层任务已经超时。建议 base_delay 0.5–1s，max_delay 5–10s，并给整个 run 设一个硬性时间窗。

### 4. 定义降级级别

把外部依赖按重要性分三级：

- **Required**：失败则整个任务失败，但需要返回结构化失败结果
- **BestEffort**：失败可跳过，Agent 继续执行后续步骤
- **Degraded**：失败后用缓存、默认值或备用 API 继续

例如天气接口可以是 BestEffort，支付接口必须是 Required，搜索接口可以是 Degraded。

### 5. 给 Agent 返回结构化错误

工具失败时不要直接 raise，而是返回一个标准错误对象，让 Agent 能继续推理：

```json
{
  "ok": false,
  "error_type": "temporary_timeout",
  "retryable": true,
  "message": "search_api timeout after 3 retries",
  "fallback_used": "cache",
  "stale": true
}
```

Agent 看到这个结构后，可以根据 `fallback_used` 和 `stale` 决定：直接用缓存结果、换关键词、跳过该步，还是告知用户“当前数据可能不是实时的”。

### 6. 幂等与检查点

对写操作，如发消息、创建订单、写数据库，一定要在调用前生成 `idempotency_key`，执行后落 checkpoint。重试前先查 checkpoint，避免重复副作用。可以用 SQLite 或 Redis 记录 `step_id -> status`。

### 7. MCP 工具连接失败处理

MCP server 崩溃或超时时，客户端需要捕获连接错误并尝试重启/重连。工具调用超时后，给 Agent 返回错误对象，而不是让整个进程退出。如果 server 重启后长连接没有自动恢复，需要显式 reset connection。

## 踩坑点

- **不要无差别重试写操作**。只对只读或带幂等键的写操作重试，否则可能重复扣款、重复发送。
- **不要捕获所有 Exception**。`KeyError`、`TypeError` 是代码 bug，不该被当成外部故障重试，否则会掩盖真实问题。
- **缓存降级容易返回过期数据**。必须标记 `stale` 和 timestamp，提醒 Agent 不要当作实时结果。
- **MCP server 重启后不一定自动重连**。很多长连接在 server 崩溃后需要重新握手，客户端要做显式 reset。
- **日志和指标要跟上**。记录 `retry_count`、`fallback_used`、`external_api_failure_total`，否则线上排障全靠猜。

## 可复用建议

- 把外部调用逻辑做成一个 `external_call` 装饰器或中间件，不要散落在各个工具里。
- 不同依赖用不同降级级别：天气接口 BestEffort，支付接口 Required + 幂等。
- 在 Agent 系统提示里明确：工具可能返回错误对象，不要盲目重试，优先使用 fallback 数据并说明来源。
- 测试用例覆盖 429、502、超时、返回 HTML 错误页、空响应，确认 Agent 不会崩。
- 监控外部 API 失败率、重试次数、降级使用率，设置告警阈值。

## 总结

外部 API 挂了是常态，Agent 的错误恢复不是把所有失败都重试，而是建立清晰的可重试边界、超时预算、降级路径和幂等检查点。工具失败时给 Agent 结构化信息，它才能做出合理决策。这样系统在部分依赖不可用时仍能体面降级，而不是整个流程崩掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/4c5da6b0be270f85.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b5a36cee0fd33bde.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/0f9f82e9a934d50b.png)

