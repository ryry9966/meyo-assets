---
title: AI Agent 容错实战：当外部 API 不可用时如何优雅降级
feedId: 31251
source: 综合讨论
publishedAt: 2026-08-01
---

# AI Agent 容错实战：当外部 API 不可用时如何优雅降级

在基于 OpenClaw 或 MCP 构建 AI Agent 的过程中，依赖外部 API 几乎是不可避免的：天气查询、搜索、代码执行、数据库操作…… Agent 将这些工具串联起来完成任务。但当某个 API 突然返回 503、连接超时或被限流时，很多 Agent 会直接抛出异常、停止工作，甚至进入无意义的死循环重试。本文分享一套务实的工程化容错方案，让 Agent 在外部 API 故障时仍能做出合理响应。

## 1. 问题：Agent 为什么会被一个挂掉的 API 绊倒？

大语言模型本身对工具调用结果有相当了容忍度，但如果工具只返回未经处理的原生错误信息，LLM 很容易误判。例如，搜索 API 超时后工具直接返回一个 Python traceback，LLM 可能得出“没有找到相关信息”的结论，却不知道这是一个临时故障。更糟糕的是，某些 Agent 框架会反复重试同一个工具，耗尽 token 和时间窗口。

因此，我们需要在工具层面建立一套“感知—重试—降级”的机制，并将处理后的结构化信息返回给 LLM，让 Agent 可以做出正确决策。

## 2. 做法：在 MCP 工具内部构建容错能力

以 OpenClaw 的 MCP server 为例，很多开发者会在自定义工具中直接调用第三方 API。下面以 `get_weather` 工具为例，说明如何引入重试、熔断和降级。

### 2.1 工具接口设计

首先约定工具返回的 JSON 结构，增加一个 `status` 字段：

```json
{
  "status": "ok",
  "data": { "temperature": 22, "condition": "晴" }
}
```

出错时：

```json
{
  "status": "error",
  "error_code": "API_UNAVAILABLE",
  "message": "天气服务暂时不可用，请稍后重试",
  "fallback": { "temperature": "未知", "condition": "无法获取" }
}
```

这样 LLM 就能根据 `status` 和 `error_code` 决定是否重试、使用 fallback 还是告知用户。

### 2.2 加入重试与退避

使用 `tenacity` 库为 API 调用添加重试逻辑，仅针对可恢复的临时错误（5xx、网络超时）：

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import requests
import time

def is_temporary_error(exception):
    if isinstance(exception, requests.exceptions.Timeout):
        return True
    if isinstance(exception, requests.exceptions.HTTPError):
        return 500 <= exception.response.status_code < 600
    return False

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception(is_temporary_error)
)
def call_weather_api(city):
    resp = requests.get(f"https://api.weather.com/{city}", timeout=5)
    resp.raise_for_status()
    return resp.json()
```

此处分三次指数退避重试（1s、2s、4s），限定了最大等待时间，避免占用 Agent 过多时间。

### 2.3 引入熔断器

当重试全部失败，且过去 60 秒内累计失败 5 次，就打开熔断器，后续请求直接返回降级数据而不再尝试真实调用，防止雪崩。可使用 `pybreaker` 实现：

```python
import pybreaker

weather_breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    timeout_duration=60
)

@weather_breaker
def get_weather_with_breaker(city):
    return call_weather_api(city)
```

熔断打开后，`get_weather_with_breaker` 抛出 `CircuitBreakerError`，我们捕获它并返回降级结果。

### 2.4 整合：工具返回有意义的状态

最终工具函数如下：

```python
def get_weather(city):
    try:
        data = get_weather_with_breaker(city)
        return {"status": "ok", "data": data}
    except pybreaker.CircuitBreakerError:
        return {
            "status": "error",
            "error_code": "API_UNAVAILABLE",
            "message": "天气服务熔断，已使用本地缓存",
            "fallback": get_cached_weather(city)
        }
    except Exception as e:
        return {
            "status": "error",
            "error_code": "UNKNOWN",
            "message": str(e)
        }
```

Agent 收到 `fallback` 后，可以在回答中说明“实时天气暂时获取不到，显示的是最近一次缓存结果”，而不是胡编或直接报错。

## 3. 踩坑记录

* **重试次数与 Agent 超时冲突**：很多 Agent 有全局任务超时（比如 120 秒），如果单个工具重试加上退避占用大量时间，后续步骤会被挤占。建议工具内部总重试时间不超过 Agent 超时的 1/3。
* **熔断器状态不持久**：默认内存存储，进程重启后失败计数清零。如果服务刚恢复就被重启，熔断阈值要从头计算，可能错失恢复窗口。可考虑用 Redis 共享状态。
* **LLM 对 fallback 字段的滥用**：在 prompt 中需要明确指示 Agent，只有遇到 `error` 状态时才使用 fallback 数据，并且要用自然语言告知用户数据来源。否则 LLM 可能无视错误直接把 fallback 当正式结果。
* **缓存时效陷阱**：本地缓存的天气如果是一小时前的，对于需要实时决策的任务可能产生误导。需要在 fallback 数据中附带 `cached_at` 时间戳，Agent 可判断是否可用。
* **幂等性风险**：重试仅适用于读操作。对于写操作（如创建订单、发送邮件），必须确保远端支持幂等，或通过唯一键去重，否则重试可能产生副作用。MCP 工具中应对写操作单独做重试控制。

## 4. 可复用的工程建议

* 把重试、熔断、降级包装成一个通用的 MCP 工具装饰器或基类，所有外部依赖工具复用。
* 在工具返回的 `error_code` 中做细粒度分类（`TIMEOUT`、`RATE_LIMITED`、`AUTH_ERROR`），方便 Agent 决定后续动作。比如遇到限流，Agent 可以主动等待几秒后再试。
* 建立统一的日志和监控：至少记录每次重试的起止时间、失败原因，便于事后了解是哪些 API 经常出问题。
* 定期演练降级流程：主动断掉一个 API，观察 Agent 的行为是否符合预期，及时调整 prompt 和容错参数。

## 5. 总结

为 Agent 赋予容错能力并不复杂，关键在于将错误从无意义的堆栈转化为结构化信息，并在工具内部完成有限次重试与优雅降级。这样一来，即使依赖的外部世界暂时崩塌，Agent 仍然可以从容地交付一个有价值的回答，而不是抛下一句“出错了”就结束。让工具坚实，Agent 才敢于走进复杂的自动化流程。

---

