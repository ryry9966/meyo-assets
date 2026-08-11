---
title: Agent 的“容错线程”：当外部 API 挂了，OpenClaw 如何实现优雅降级
feedId: 32533
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

你的 Agent 昨晚还正常应答，今天一早就开始连续失败。排了半天才发现是调用的第三方天气 API 全线返回 503。用户看到的是“抱歉，我无法处理”，你看到的是日志里一堆 `FetchError: request to https://api.example.com failed`。

在 OpenClaw 生态里，Agent 往往依赖多个 MCP 服务或自定义插件来访问外部世界：翻译、搜索、汇率、股票、甚至是内部微服务。这些外部依赖不可能永远在线，API 限流、网关超时、上游宕机都是常态。

如果我们不加任何错误恢复机制，Agent 的任务流程就是一条脆弱的直线——一个工具调用失败，整个链路中断，用户只能得到冷冰冰的出错信息。工程化实践要求我们给 Agent 装上一套“容错线程”：重试、熔断、降级兜底。

下面就以一个 OpenClaw 调用 MCP 天气服务的场景为例，铺开三种可落地的错误恢复策略。

## 问题定义

假设我们有一个 MCP 工具 `get_weather`，底层调用第三方天气 API，返回城市天气信息。Agent 在处理用户“明天上海会下雨吗”的请求时，会调用该工具。当上游服务出现以下问题时：

- 瞬时网络抖动导致超时
- API 短暂限流（429）
- 上游服务宕机返回 5xx

Agent 应当：
1. 尽可能自动恢复，不让用户感知瞬时故障
2. 无法恢复时给出有意义的人话降级回复，如“暂时拿不到最新天气，但我记得上次查询是晴天”
3. 避免因重试而进一步压垮已不堪重负的上游

## 做法与步骤

### 1. 带指数退避的重试

最基础的手段就是在 MCP 工具内部对网络请求加重试。注意必须是幂等的 GET 请求，POST 等写操作需谨慎。

以 Python 实现的 `weather_mcp_server` 为例，核心逻辑如下：

```python
import httpx
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    reraise=True
)
async def fetch_weather(city: str):
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.get(f"https://api.weather.com/v1/{city}")
        resp.raise_for_status()
        return resp.json()
```

`tenacity` 库提供了声明式重试能力，指数退避避免雷同的时间间隔造成“惊群效应”。三次重试总共约 10 秒左右，用户体感尚可接受。

在 MCP 工具函数中调用 `fetch_weather`，捕获异常后抛出标准 `ToolError`，让 OpenClaw 框架感知工具失败。

### 2. 断路器避免持续无效请求

如果上游已经确认宕机，反复重试就是在浪费资源和用户耐心。电路断路器（Circuit Breaker）可以快速失败，为降级争取时间。

在 OpenClaw 环境下，断路器状态可以放在 MCP 工具模块的全局字典里，也可用 Redis 实现跨会话持久化。简易实现：

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=30):
        self.failure_count = 0
        self.last_failure_time = None
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout

    def call(self, func, *args, **kwargs):
        if self.last_failure_time and \
           (time.time() - self.last_failure_time) < self.recovery_timeout:
            raise Exception("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            self.failure_count = 0
            return result
        except Exception:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self.last_failure_time = time.time()
            raise
```

将 `fetch_weather` 包装进 `circuit_breaker.call()`，一旦短期内连续失败 3 次，之后 30 秒内直接异常，不再重试。Agent 捕获到断路器打开异常后，立即走降级逻辑。

### 3. 缓存兜底与降级回复

当 API 完全不可达时，最后一道防线就是返回上次成功的缓存数据，并附带时间信息。

```python
weather_cache = {}  # 简单内存缓存，生产可用 Redis

def get_weather_with_fallback(city: str):
    try:
        data = fetch_weather(city)
        weather_cache[city] = {"data": data, "ts": time.time()}
        return data
    except Exception:
        cached = weather_cache.get(city)
        if cached:
            age_min = int((time.time() - cached["ts"]) / 60)
            return {
                "source": "cache",
                "cached_age_minutes": age_min,
                "data": cached["data"]
            }
        raise
```

Agent 侧收到 `source: "cache"` 后，回复模板变为：

> “目前天气服务暂时连接不上，我查到了 {cached_age_minutes} 分钟前的数据：当时上海是晴天，气温 15°C，仅供参考。”

这种降级既诚实又有温度，同时不中断对话流程。

## 踩坑点

1. **重试非幂等接口**：对 POST/PUT 接口重试可能导致重复写入。可限定重试仅用于 GET，或在服务端实现幂等键。
2. **断路器状态不持久**：进程重启后计数清零，若另一个实例已经触发熔断但没通知其他实例，仍会向外发送请求。建议用 Redis 集中管理状态，并设置合理的 TTL。
3. **雪崩式重试**：多个 Agent 实例同时重试，上游刚恢复就被打挂。重试策略要加入随机抖动（jitter），并限制集群级别并发重试数。
4. **缓存数据误导用户**：过期太久的缓存可能完全错误。建议设置最大缓存有效期，超过后直接告知用户“暂时无法服务”，而不是给出虚假信息。
5. **日志与告警缺失**：降级为静默处理让人忽略问题严重性。每次进入降级都应记录 WARN 日志，并触发监控告警，以便排障和上游协调。

## 可复用的工程化建议

不要在每个 MCP 工具里各自实现一遍。比较理想的形态是封装一个统一的“弹性调用”装饰器或中间件，供所有 MCP 工具复用：

```python
def resilient_call(cache_key=None, max_retries=3, use_circuit=True):
    def decorator(func):
        # 组合重试、断路器、缓存逻辑
        ...
        return wrapper
    return decorator
```

然后在 MCP 工具定义时一行装饰即可：

```python
@mcp.tool()
@resilient_call(cache_key="weather_{city}", max_retries=2)
def get_weather(city: str) -> dict:
    ...
```

如果团队已采用 OpenClaw 插件架构，还可以将整个弹性处理逻辑做成一个中间件插件，在框架层面对所有工具调用统一注入错误恢复策略，进一步降低认知负载。

此外，建议定期做“混沌”测试：手动断开某个 MCP 服务，观察 Agent 行为和日志，验证降级链路是否按预期工作。

## 总结

AI Agent 最终是一个运行在真实网络里的软件系统，而不是实验室里的完美环境。重试、熔断、缓存兜底这三板斧并不新鲜，但恰恰是工程化的基础。在 OpenClaw 这类 Agent 框架中，我们可以从小处着手：先为一个 MCP 工具加上重试和降级，验证效果后再抽象为可复用模块。用户的耐心会在每一次“抱歉，我暂时连不上，但之前看到是这样...”中延续，而不是在一次“系统故障”中流失。

---

