---
title: 当 MCP 工具背后的 API 挂了：给 Agent 加上可恢复的韧性层
feedId: 31919
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：Agent 对外部 API 的依赖比你想象的更脆弱

在 OpenClaw 这类以工具调用为核心的 Agent 框架中，MCP 工具往往直接封装了一个远程 API。天气查询、股票数据、翻译服务、知识库检索——这些能力背后都是 HTTP 调用。一旦外部服务不可用，Agent 不会“知道”如何处理，通常只会把原始异常堆栈扔给 LLM，然后任务中断，整个自动化流水线卡住。

多数教程只教你怎么写工具，却没教你怎么让工具“坏得优雅”。本文总结了一套在 MCP 工具层构建错误恢复机制的工程实践，目标是在 API 抖动或短期宕机时，让 Agent 尽量自愈，而不是直接撂挑子。

## 问题拆解：API 失效的三种典型场景

1. **瞬时故障**：网络闪断、DNS 解析偶发失败、服务端 502/503，通常几秒后就能恢复。
2. **过载保护**：API 触发限流，返回 429 或 connection reset，需要等待一段时间再试。
3. **持续不可用**：服务宕机超过数分钟，或凭据过期、接口废弃等永久性错误。

Agent 的错误恢复策略需要区分这三类，而不是不分青红皂白重试或直接失败。

## 做法：在工具函数中嵌入韧性层

以下所有实践均围绕 MCP 工具或普通 Python 异步函数展开，不依赖特定框架，但可以直接用于 OpenClaw 的插件/工具定义。

### 1. 超时与重试：用指数退避避免踩踏

最基础也最容易被忽略。`httpx` 或 `aiohttp` 的默认超时往往太长，导致一个慢请求卡住整个 Agent 循环。建议显式设置连接超时和读取超时，并为可重试的错误实现指数退避重试。

```python
import asyncio
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

RETRYABLE_STATUSES = {429, 502, 503, 504}

def is_retryable(exception):
    if isinstance(exception, httpx.HTTPStatusError):
        return exception.response.status_code in RETRYABLE_STATUSES
    if isinstance(exception, (httpx.TimeoutException, httpx.NetworkError)):
        return True
    return False

@retry(
    retry=retry_if_exception_type(Exception),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry_error_callback=lambda retry_state: None
)
async def call_external_api(payload: dict) -> dict:
    async with httpx.AsyncClient(timeout=httpx.Timeout(5.0, read=10.0)) as client:
        resp = await client.post("https://api.example.com/data", json=payload)
        resp.raise_for_status()
        return resp.json()
```

- **踩坑点**：`tenacity` 的 `retry` 默认会捕获所有异常，一定要配合 `retry_if_exception_type` 或自定义条件，否则连 `KeyboardInterrupt` 都会重试。另外，如果工具调用有副作用（如创建资源），必须保证幂等性，否则重试可能创建多条重复数据。

### 2. 熔断器：防止雪崩效应

当 API 连续失败时，继续重试只会让下游压力更大，也拖慢 Agent 自身的速度。引入一个简单的熔断器：统计最近 N 次调用的失败率，超过阈值则短时间内直接快速失败，避免无效等待。

可以使用 `pybreaker` 库或手搓一个基于时间的计数器。在 MCP 工具的环境下，每个工具实例可以维护自己的熔断状态，通过闭包或模块级变量实现。

```python
from pybreaker import CircuitBreaker

breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@breaker
async def tool_call_with_breaker(payload: dict):
    return await call_external_api(payload)
```

- **踩坑点**：熔断器的 timeout 期过后会进入“半开”状态，允许一次探测调用。如果探测仍然失败，重新熔断。要保证探测调用也受超时控制，否则会拖长恢复时间。另外，在 Agent 场景下，短时间的大量工具调用可能让熔断器统计失真，最好按工具+目标 API 分别设置实例。

### 3. 降级策略：不要让 LLM 对着异常发呆

当所有重试手段都用完，API 仍不可用时，工具函数不应该抛出原始异常。那样 LLM 会收到一堆 traceback，很可能给出“我无法处理这个错误”的无用回复。更好的做法是返回一个结构化的降级结果，让 LLM 能够基于此继续规划。

降级策略的几个层次：
- **返回缓存数据**：在上一次成功调用时将结果缓存（TTL 可长一些），失败时先查缓存。
- **返回空数据 + 明确标识**：例如 `{"error": "TEMPORARILY_UNAVAILABLE", "cached_age": 120}`，LLM 可以判断是否要等待、用旧数据、或跳过该步骤。
- **替代信息来源**：如主 API 挂了，改查本地静态数据库或更低优先级的免费 API。

我们在工具函数中统一包装返回：

```python
async def safe_tool_call(payload):
    try:
        return await tool_call_with_breaker(payload)
    except Exception:
        cached = await cache.get(payload)
        if cached:
            return {**cached, "_source": "cache", "_stale": True}
        return {"error": "SERVICE_UNAVAILABLE", "message": "外部服务暂时不可用，请跳过此部分或使用旧数据"}
```

- **踩坑点**：降级返回的数据格式必须与正常返回结构兼容，否则 LLM 可能解析失败。建议在工具的描述 schema 里注明可能返回的降级字段，或者让工具在降级时返回人类可读的提示，如“天气数据暂时无法获取，上一次查询结果是在5分钟前，仅供参考”。

### 4. 让 Agent 感知错误并重规划

更进一步的韧性是让 Agent 的规划器（ReAct 或 Tree-of-thought）在收到降级信号后主动调整行为。可以在工具返回中加入一个特殊字段 `_action_hint: "retry_later"` 或 `"use_fallback"`，在 OpenClaw 的自定义执行循环中检查这个 hint，触发任务队列的重排或跳过当前步骤。这需要在 Agent 框架层面做少量修改，但能极大提升自动化流程的健壮性。

## 可复用建议与总结

- **封装一个 `resilience` 装饰器或工具包装器**：把所有重试、超时、熔断、降级逻辑收敛到一个函数工厂，后续所有 MCP 工具只需传参即可获得韧性能力。
- **日志与监控不可少**：记录每次重试、熔断状态变化、降级命中，便于事后分析第三方 SLA 和调整参数。
- **测试你的降级路径**：用 toxiproxy 或直接 mock HTTP 层来模拟超时、503、连接拒绝，确认 Agent 在这些场景下的行为符合预期，而不是卡死或无限循环。
- **区分可恢复与永久性错误**：4xx 请求错误（除了 429）一般没必要重试，应直接返回错误给 LLM，节省时间。

API 总会在最不合适的时候挂掉。给 Agent 的工具加上这些恢复机制，不是为了写出永远不会失败的系统，而是为了让失败变得可管理、可自愈。在自动化实践中，一个能优雅降级的 Agent，远比一个只在晴天才工作的 Agent 有价值。

---

