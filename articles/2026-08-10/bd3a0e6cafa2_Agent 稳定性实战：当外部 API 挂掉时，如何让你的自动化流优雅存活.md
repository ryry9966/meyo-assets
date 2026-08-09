---
title: Agent 稳定性实战：当外部 API 挂掉时，如何让你的自动化流优雅存活
feedId: 32314
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：Agent 的脆弱时刻

在基于 OpenClaw 的 Agent 工作流里，外部 API 调用几乎是每条链路的必经节点。无论是通过 MCP 工具访问第三方数据源、调用 LLM 推理服务，还是触发 Webhook 通知，Agent 的“智能”本质上依赖一连串不可控的远端响应。

一旦某个外部 API 宕机、限流或返回畸形数据，Agent 面临的不是单纯报错，而是整个任务链断裂、上下文丢失、重试风暴甚至引发雪崩。尤其是在长时间运行的自动化任务中（如定时巡检、批量数据处理），一次偶发的 HTTP 503 可能让 5 小时的任务前功尽弃。

社区里常见的声音是“加个重试就好了”，但在工程实践中，错误恢复远比 `for i in range(3)` 复杂。本文从 OpenClaw 插件开发者的视角，梳理一套可落地的 API 故障恢复机制，不依赖特定云服务，用标准库与通用模式解决问题。

## 问题拆解：故障的脸谱

外部 API 失效通常表现为三类：

1. **瞬时故障**：网络抖动、服务重启导致的短暂不可用，HTTP 5xx、连接超时。
2. **速率限制**：API 返回 429 Too Many Requests，需要退避等待。
3. **逻辑故障**：返回 4xx（除了 429），如认证过期、参数错误。这类错误重试无效，需要快速失败并告警。

合理的设计必须区分故障类型，否则会浪费资源在无效重试上，或在不该放弃时过早投降。

在 OpenClaw 插件中，一个典型场景：Agent 通过 MCP 客户端调用某个搜索工具，该工具封装了外部搜索 API。搜索 API 偶尔 503，但绝大部分时间正常。我们需要让 Agent 在遇到此类错误时自动恢复，同时不阻塞主流程。

## 做法与步骤

### 1. 为 MCP 工具封装带智能策略的调用层

不让原始 MCP 工具直接暴露给 Agent 执行器，而是在其外包裹一层 `ResilientClient`。核心逻辑：

- 拦截所有 HTTP 异常（如 `httpx.HTTPStatusError`）
- 基于状态码和自定义配置决定行为

以 Python 实现为例（适配 OpenClaw 插件常用技术栈）：

```python
import asyncio
import logging
from typing import Any, Callable, Optional
import httpx

logger = logging.getLogger(__name__)

class APICallConfig:
    max_retries: int = 3
    base_delay: float = 1.0
    max_delay: float = 30.0
    retryable_statuses: set[int] = {429, 500, 502, 503, 504}
    fatal_statuses: set[int] = {400, 401, 403, 404}  # 快速失败

async def resilient_call(
    fn: Callable[..., Any],
    *args,
    config: Optional[APICallConfig] = None,
    **kwargs
) -> Any:
    cfg = config or APICallConfig()
    last_exception = None
    delay = cfg.base_delay

    for attempt in range(cfg.max_retries + 1):
        try:
            return await fn(*args, **kwargs)
        except httpx.HTTPStatusError as e:
            status = e.response.status_code
            if status in cfg.fatal_statuses:
                logger.error(f"Fatal API error {status}, aborting: {e}")
                raise
            if status not in cfg.retryable_statuses and status >= 400:
                raise  # 未知 4xx 也直接失败

            if attempt == cfg.max_retries:
                logger.error(f"Max retries reached for status {status}")
                raise

            # 指数退避 + 随机抖动
            sleep_time = min(delay * (2 ** attempt), cfg.max_delay)
            sleep_time *= (0.5 + 0.5 * random.random())
            logger.warning(f"Retryable error {status}, retrying in {sleep_time:.2f}s (attempt {attempt+1}/{cfg.max_retries})")
            await asyncio.sleep(sleep_time)
            last_exception = e

    if last_exception:
        raise last_exception
```

### 2. 在 MCP 工具定义中集成恢复逻辑

假设你要编写一个封装了 Google 搜索的 MCP 工具，可在工具的 `call_tool` 回调中直接使用上述 `resilient_call`：

```python
from openclaw_mcp import MCPServer, tool

search_server = MCPServer("search")

@tool()
async def search_web(query: str) -> str:
    async def _do_request():
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.get("https://api.search.example.com/v1/search",
                                    params={"q": query})
            resp.raise_for_status()
            return resp.json()["results"]

    try:
        results = await resilient_call(_do_request, config=APICallConfig(max_retries=2))
        return format_results(results)
    except Exception as e:
        # 所有恢复手段用尽，返回降级结果
        return fallback_search(query, error=str(e))
```

### 3. 设计降级（Fallback）策略

重试只是恢复的第一道防线，更关键的是兜底逻辑。根据业务可选择：

- **缓存兜底**：使用 `cachetools` 或本地文件缓存最近成功的查询结果。故障时先查缓存，缓存未命中再返回预置的静态响应。
- **功能降级**：如果搜索 API 不可用，回退到本地文本匹配或较低精度的离线索引。
- **优雅占位**：在 Agent 的最终回复中诚实告知“部分信息暂时不可用”，并标记任务状态为 `partial_success`，避免误导下游。

实现缓存示例：

```python
from cachetools import TTLCache
cache = TTLCache(maxsize=100, ttl=600)

def fallback_search(query: str, error: str) -> str:
    cached = cache.get(query)
    if cached:
        logger.info(f"Using cached result for query '{query}'")
        return f"[Cache] {cached}"
    return f"Search unavailable: {error}"
```

### 4. 熔断与状态监控

当重试策略持续生效时，可能说明上游服务已深度故障。可以引入简单的熔断器，避免饿死 Agent 线程池。利用 `pybreaker` 或自己实现：

```python
class CircuitBreaker:
    def __init__(self, fail_threshold=5, recovery_timeout=60):
        self.fail_count = 0
        self.fail_threshold = fail_threshold
        self.recovery_timeout = recovery_timeout
        self.last_fail_time = 0.0
        self.state = "CLOSED"

    async def call(self, fn, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_fail_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker OPEN, rejecting call")

        try:
            result = await fn(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.fail_count = 0
            return result
        except Exception:
            self.fail_count += 1
            self.last_fail_time = time.time()
            if self.fail_count >= self.fail_threshold:
                self.state = "OPEN"
                logger.critical("Circuit breaker OPEN due to consecutive failures")
            raise
```

并将熔断器放在 `resilient_call` 外层，既能保护上游调用频率，也能防止 Agent 不断在无效工具上浪费算力。

## 踩坑点

1. **无差别的重试放大流量**  
   熔断未开启时，多个 Agent 实例可能同时发起重试，在 API 恢复瞬间形成“惊群”。务必全局共享熔断状态（例如通过 Redis 实现分布式熔断），并限制全局并发重试数。

2. **重试导致的数据不一致**  
   对于非幂等的写操作（如创建订单），重试可能造成重复提交。必须在工具层保证接口的幂等性（携带 `idempotency-key`），或对写操作禁用自动重试。

3. **缓存穿透风险**  
   恶意或异常的高频查询在 API 故障期间仍可能打满本地缓存，需设置缓存大小和 TTL，并与熔断器联动。

4. **错误信息丢失**  
   多轮重试后，原始错误堆栈可能被覆盖。日志需记录每次尝试的详细上下文，包括请求ID、时间、延迟，便于事后排障。使用 `rich` 或结构化日志输出。

5. **超时控制不当**  
   `resilient_call` 的总超时时间应小于 Agent 节点的任务超时，否则任务会被提前取消，留下僵尸上下文。建议分别设置单次请求超时和累计重试超时。

## 可复用建议

- **配置化**：将重试次数、退避参数、熔断阈值放入环境变量或配置中心，可在运行时调整，无需重新部署插件。
- **统一中间件**：为所有基于 MCP 的出站 HTTP 调用编写一个共享的 `AsyncClient` 传输层，集成重试和熔断，避免每个工具重复实现。
- **健康检查端点**：在插件内部暴露一个简单的 HTTP 健康端点，返回所有依赖的外部 API 状态。配合 OpenClaw 监控面板，可以快速定位故障源。
- **模拟故障演练**：使用 `toxiproxy` 或自定义代理人为注入延迟、错误，验证恢复流程在上线前是可靠的。

## 总结

AI Agent 的错误恢复本质是在不确定性中构建有限的确定性。重试、熔断、降级、缓存这四种模式组合出的弹性，远比依赖单一 API 的 SLA 实在。在 OpenClaw 插件生态中，把恢复逻辑下沉到工具调用层，Agent 本身无需感知故障细节，就能保持任务流的健壮。

稳定的自动化不是从来不出错，而是出错了能被兜住、被理解、被修复。下一次当你在凌晨被报警吵醒，至少可以放心，Agent 已经自行撑过十分钟的 API 抖动，只留下一条清晰的恢复日志。

---

