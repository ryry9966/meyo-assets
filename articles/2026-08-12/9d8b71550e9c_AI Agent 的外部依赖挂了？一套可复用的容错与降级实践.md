---
title: AI Agent 的外部依赖挂了？一套可复用的容错与降级实践
feedId: 32687
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：Agent 的“阿喀琉斯之踵”

在基于 OpenClaw 或 MCP 构建自动化 Agent 时，我们越来越依赖外部 API：大模型推理、搜索引擎、数据库、第三方服务。这些依赖一旦不可用，原本流畅的 Agent 链路就会直接抛异常，甚至导致整条 Pipeline 中断。

更棘手的是，Agent 的上下文往往有状态。如果一次调用失败后不加以处理，后续步骤拿到的仍是错误返回值，可能引发“雪崩”——比如把错误信息当作正常数据塞进 prompt，让模型产生幻觉。

我们需要一种工程化的错误恢复机制，既不破坏 Agent 的自主决策能力，又能防止单点故障蔓延。这篇文章分享一套我在 OpenClaw 环境中的实践，涵盖重试、熔断、降级和回退，适用于任何调用外部 API 的 MCP 工具或插件。

## 问题拆解

外部 API 挂掉的表现五花八门：连接超时、DNS 解析失败、HTTP 5xx、限流 429、响应体格式错误。从 Agent 视角看，需要解决三个层次的问题：

1. **检测**：如何区分临时故障和永久故障？
2. **响应**：是重试、短路还是给一个兜底值？
3. **状态恢复**：失败后如何让 Agent 继续执行剩余步骤，且不丢失关键上下文？

很多开发者直接在工具函数里 try-except，再不管三七二十一 `return None`，这会让 Agent 在不知情的情况下继续跑，把 null 当数据用。这是最危险的模式。

## 做法与步骤

我在 OpenClaw 项目中构建了一个轻量的可插拔容错层，核心是三个组件：**安全调用装饰器**、**熔断器状态机**、**降级策略注册**。

### 1. 安全调用装饰器

```python
import functools
import asyncio
import logging
from typing import Callable, Any

logger = logging.getLogger("agent.fallback")

def safe_mcp_call(
    max_retries: int = 3,
    base_delay: float = 0.5,
    backoff_factor: float = 2.0,
    fallback_value: Any = None,
):
    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_exc = e
                    if not _is_retryable(e):
                        break  # 非临时错误直接跳出
                    delay = base_delay * (backoff_factor ** (attempt - 1))
                    logger.warning(
                        f"Retry {attempt}/{max_retries} for {func.__name__}: {e}. "
                        f"Waiting {delay:.1f}s"
                    )
                    await asyncio.sleep(delay)
            # 所有重试耗尽或非临时错误
            logger.error(f"All retries exhausted for {func.__name__}: {last_exc}")
            return fallback_value
        return wrapper
    return decorator

def _is_retryable(exc: Exception) -> bool:
    # 根据实际异常类型判断，如 timeout、503、429
    if isinstance(exc, asyncio.TimeoutError):
        return True
    if hasattr(exc, "status_code"):
        return exc.status_code in (429, 500, 502, 503, 504)
    return False
```

将它用在 MCP 工具定义上：

```python
@mcp.tool()
@safe_mcp_call(max_retries=3, fallback_value={"results": [], "status": "degraded"})
async def search_web(query: str) -> dict:
    ...
```

注意：`fallback_value` 要和正常返回结构一致，避免下游解析报错。这里返回一个空结果加状态标记，Agent 可根据 `status` 字段决定是否走备用逻辑。

### 2. 熔断器

重试只能解决短期抖动。如果下游服务长时间不可用，反复重试只会增加延迟，可能让 Agent 的 HTTP 连接池耗尽。引入简单的熔断器：连续失败超过阈值，直接快速失败，一段时间后半开尝试。

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "closed"  # closed / open / half_open

    def before_call(self):
        if self.state == "open":
            if (time.monotonic() - self.last_failure_time) > self.timeout:
                self.state = "half_open"
            else:
                raise CircuitOpenError("Circuit breaker is open")
        # 其他状态允许通过

    def on_success(self):
        self.failure_count = 0
        self.state = "closed"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.monotonic()
        if self.failure_count >= self.failure_threshold:
            self.state = "open"
```

与装饰器结合，在重试前检查熔断器状态。如果熔断器打开，直接返回降级值，不再尝试。

### 3. 降级策略与 Agent 决策链路

降级不只是返回空值，还包括调用备用 API、从缓存读取、使用规则计算等。可以维护一个降级策略注册表：

```python
BACKUP_STRATEGIES = {
    "search_web": lambda q: run_local_search(q),
    "get_stock_price": lambda symbol: cached_price(symbol) or {"price": -1, "source": "fallback"},
}
```

在 `safe_mcp_call` 里增加 `fallback_strategy` 参数，允许传入可调用对象。熔断打开或重试耗尽时，优先执行该策略。

Agent 层面，可以通过观察工具返回内容中的 `status` 或特定字段，触发备用路径。例如在 OpenClaw 的 prompt 中约定：如果看到 `"status": "degraded"`，自动降低本次回答的置信度，并明确告知用户部分数据不可用。

### 4. 融入 OpenClaw 生命周期

在 OpenClaw 中，Agent 以任务队列方式运行，可以注册全局错误钩子。我在插件入口处统一包装所有对外工具：

```python
def wrap_all_tools(tool_dict):
    for name, func in tool_dict.items():
        tool_dict[name] = safe_mcp_call(
            max_retries=2,
            fallback_value=_default_fallback(name)
        )(func)
    return tool_dict
```

然后通过插件加载时 apply，避免在业务代码中散落处理逻辑。

## 踩坑点

- **超时设置层级混乱**：有的在 HTTP client 设了超时，又在重试层设了总超时，可能导致重试仍被外层超时中断。建议统一用 `asyncio.timeout` 或 CancellationToken 管理，保留一次调用的合理时长，重试应在内部消化。
- **无差别重试**：非幂等写操作重试可能造成重复扣款、重复发送消息。这类工具应标为 `retriable=False`，直接失败让 Agent 决定下一步。
- **降级值过于“无害”**：`return None` 或 `return {}` 没有痕迹，Agent 容易无视。建议给降级值带上元数据，让 Agent 能感知到发生了降级。
- **熔断指标放在进程内**：如果 Agent 是多实例部署，内存熔断器会导致每个 Pod 独立计数，可能大多数 Pod 处于 closed 而实际下游已不可用。生产环境需用 Redis 等集中式熔断状态存储。
- **忘记了 CircuitHalfOpen 的探测逻辑**：半开状态应允许少量请求探测，如果仍失败要重新打开，不要无脑清空计数。

## 可复用建议

1. **把容错逻辑做成插件或中间件**，而非散落在每个工具函数中。OpenClaw 的插件机制很适合注入统一的 `invoke` 包装。
2. **为每个外部依赖建立健康检查 Sidecar**（例如定时 HEAD 请求），通过 shared state 供所有工具函数读取，可以提前减速，避免等到真实调用时才发现。
3. **在 Agent prompt 中显式告知降级可能性**。比如：“当工具返回 `status: degraded` 时，你需要告知用户数据可能不全，并建议重试时间。不要捏造数据。”
4. **可观测性**：重试、熔断、降级事件都要打点，用 log + metric 记录，便于发现哪些 API 是稳定性短板。

## 总结

外部 API 挂掉不是“会不会”的问题，而是“什么时候”的问题。AI Agent 的特殊性在于，它不是一个简单的请求响应服务，而是一条有状态的决策链。我们的容错设计不能只停留在技术层面的重试和兜底，更要考虑如何让 Agent 在语义上理解“当前环境不可靠”，并做出合理退让。

在 OpenClaw 环境下，通过装饰器 + 熔断器 + 降级注册表的组合，我们可以把错误恢复能力做成一个可以复用的基础设施层，让 Agent 工具开发更关注业务逻辑。这套方案代码量不大，但能有效避免半夜被 PagerDuty 叫醒。

---

