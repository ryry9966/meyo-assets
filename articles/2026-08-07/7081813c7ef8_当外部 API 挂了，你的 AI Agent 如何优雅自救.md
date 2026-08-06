---
title: 当外部 API 挂了，你的 AI Agent 如何优雅自救
feedId: 31941
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：API 迟早会挂，Agent 不能跟着殉情

在 OpenClaw 这类自动化 Agent 里，外部 API 是核心“感官”之一 —— 翻译、天气、数据库查询、模型推理，几乎所有工具调用最终都落到 HTTP API 上。但网络抖动、第三方限流、证书过期、机房断电……这些事总会在你意想不到的时候发生。一个在没有容错设计的 Agent 里，一次简单的 `requests.get` 超时，就可能导致整个任务链中断，而用户得到的只是一个冰冷的 traceback。

我们真正需要的是一个能“自己站起来”的 Agent：短暂故障重试，持续故障降级，同时对下游保持透明。

## 问题拆解：别把失败当作一种失败

API 失败不是单一事件，工程上至少要区分：

- **瞬时故障**（503、网络超时、`Connection reset`）—— 值得重试；
- **持续性故障**（404、401、连续 5xx 超过阈值）—— 重试只会加重负担；
- **限流**（429）—— 等待 `Retry-After` 或指数退避；
- **逻辑错误**（业务状态码异常）—— 需降级或提示用户。

这些场景无法用一个 `while True: try again` 解决。

## 做法与步骤：从工具调用到韧性闭环

以下以 OpenClaw 典型 `tool` 插件为例（Python 环境），展示如何给外部 API 增加一套“错误恢复”包装，每一步都可以复制到你的现有工具中。

### 1. 错误分类与隔离

在工具函数内部，不再直接 `response.raise_for_status()`，而是精细分类：

```python
import requests
from requests.exceptions import Timeout, ConnectionError

def classify_error(e):
    if isinstance(e, Timeout):
        return "timeout"
    if isinstance(e, ConnectionError):
        return "network"
    resp = getattr(e, 'response', None)
    if resp is not None:
        if resp.status_code == 429:
            return "rate_limit"
        if 500 <= resp.status_code < 600:
            return "server_error"
        if resp.status_code in (404, 401, 403):
            return "client_error"
    return "unknown"
```

### 2. 智能重试（指数退避 + 抖动）

只对 `timeout`、`network`、`server_error`、`rate_limit` 做重试。利用 `tenacity` 库避免自己写 bug：

```python
from tenacity import (
    retry, stop_after_attempt, wait_exponential, retry_if_exception
)

def is_recoverable(exception):
    return classify_error(exception) in ("timeout", "network", "server_error", "rate_limit")

@retry(
    retry=retry_if_exception(is_recoverable),
    wait=wait_exponential(multiplier=1, min=1, max=30) + wait_random(0, 2),
    stop=stop_after_attempt(4),
    reraise=True
)
def call_external_api(url, payload):
    resp = requests.post(url, json=payload, timeout=5)
    if resp.status_code == 429:
        raise Exception("rate_limit")
    resp.raise_for_status()
    return resp.json()
```

要点：**一定要限制重试次数和最大等待时间**，否则一个卡死的第三方会让你工具线程池耗尽。

### 3. 熔断器

重试 4 次仍失败，也许对方已经进入长时间恶化。熔断器能快速失败，避免请求堆积。使用 `pybreaker` 清晰可见：

```python
import pybreaker

breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=30)

def api_with_breaker(url, payload):
    try:
        return breaker.call(call_external_api, url, payload)
    except pybreaker.CircuitBreakerError:
        # 熔断开启，直接走降级
        return get_fallback_result(url)
    except Exception:
        return get_fallback_result(url)
```

当连续失败 5 次后，接下来 30 秒内所有调用直接抛 `CircuitBreakerError`，避免雪崩。

### 4. 缓存与降级

缓存成功结果，作为降级的第一梯队；没有缓存时，返回预设的静态数据或标记性的空结果，并在返回中带上 `source="fallback"` 字段。

```python
from cachetools import TTLCache

cache = TTLCache(maxsize=128, ttl=600)  # 10 分钟缓存

def get_with_cache(url, payload):
    cache_key = url + str(payload)
    if cache_key in cache:
        return cache[cache_key], "cache"
    try:
        result = api_with_breaker(url, payload)
        cache[cache_key] = result
        return result, "live"
    except Exception:
        if cache_key in cache:
            return cache[cache_key], "stale_cache"
        return {"error": "unavailable", "data": None}, "fallback"
```

在 OpenClaw 工具的返回中，Agent 可以依据 `source` 字段调整回复：“当前使用的是 10 分钟前的缓存数据”，用户感知更透明。

### 5. 监控闭环

工具层面嵌入统一日志，记录失败原因、来源、耗时，方便后续告警（可以接入你已有的通知 channel，如钉钉/飞书或 OpenClaw 的内置事件）：

```python
import logging
logger = logging.getLogger("tool_resilience")

# 在 get_with_cache 的异常分支添加
logger.warning(f"API {url} degraded, reason={reason}, source={source}")
```

## 踩坑点

- **重试风暴**：多个 Agent 实例同时重试会放大流量，把半亡的 API 彻底打死。考虑全局并发限流或共享断路器状态。
- **幂等性破坏**：扣费、下单类 API 若重试未做幂等处理，会造成重复操作。对非只读请求，要么要求下游提供幂等键，要么只允许一次重试。
- **缓存穿透与旧数据**：TTL 太长，用户看到过期信息；TTL 太短，起不到保护作用。按业务可接受延迟来设定，比如天气 30 分钟，汇率 10 分钟。
- **熔断恢复时机**：`reset_timeout` 太短，半开状态会被连续失败打回；太长则影响恢复速度。建议从 30 秒起步，结合监控调优。
- **降级不彻底**：返回 `None` 或空列表时，上层 Agent 如果没做判空，可能继续抛错。降级数据要和成功返回格式严格一致。

## 可复用建议

将这些逻辑沉淀为一个 `ResilientApiTool` 装饰器或工具工厂，通过配置参数（重试次数、断路器阈值、缓存 TTL、降级数据）即可包装任何外部 API 调用。如果你已在使用 MCP 协议，可以把这套能力做成一个 **Resiliency MCP Server**，让多个 Agent 共享同一个容错层，避免重复实现。

最后，始终给 Agent 的 prompt 中注入一条上下文：“当工具返回 `source` 为 `fallback` 或包含 `error` 时，请向用户解释当前部分功能受限，并提供替代操作。” 这样整个闭环才真正通向用户友好。

## 总结

外部依赖的不可靠是分布式系统的常态，不要试图让 API 永远不挂，而是让你的 Agent 在 API 挂掉时依然体面地站立 —— 通过重试、熔断、缓存、降级、监控这一套组合拳，使自动化流程具备工程韧性。把错误恢复从“意外分支”变成一个可配置、可观测的标准件，才是面向生产的 Agent 设计。

---

