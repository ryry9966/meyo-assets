---
title: AI Agent 的错误恢复实战：当外部 API 挂掉之后
feedId: 32230
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：Agent 的“阿喀琉斯之踵”

在 OpenClaw 社区，大家已经习惯通过 MCP 或插件将各类外部 API 封装为 Agent 的工具：查天气、搜文档、取股价、调企业内部接口……整个自动化流程跑得很顺畅，直到某个依赖方突然挂了。

外部 API 不可用并不是偶发。网络抖动、目标服务重启、限流、凭证过期，这些都会让原本利落的工具调用当场炸开。如果没有事先设计恢复逻辑，Agent 会直接把异常堆栈吐给用户，或者更糟——在自动化的 Pipeline 里静默失败，下个节点拿着空数据继续跑，产出脏结果。

## 问题拆解：只靠 try-catch 不够

很多团队在工具函数里加一层 try-catch，捕获异常后返回 `"API 调用失败"`。但这样带来了几个新麻烦：

- **Agent 不理解失败语义**：大模型收到 `"API 调用失败"`，可能会自行脑补错误原因，甚至开始幻觉数据。
- **重试乱用**：对所有异常无差别重试，碰上 404 还猛打，既浪费资源又容易触发限流。
- **失败传播**：在一次会话中，某个工具失败可能导致 Agent 多次重新触发同一个调用，陷入死循环，烧 token 还不解决问题。
- **用户困惑**：最终用户看到的是“抱歉，我无法完成”，却不知道是暂时还是永久故障。

所以，我们要做的是在**工具层**和 **Agent 决策层**两个平面上同时设计错误恢复，形成一套可以复用的工程模式。

## 做法与步骤

下面给出的方案基于 OpenClaw 的工具注册机制，但同样适用于任何用 Python 编写的 Agent 工具。

### 1. 让错误结构化，而不是字符串

工具函数不应该返回纯文本错误。定义标准的错误响应模型，给 Agent 一个结构化的“信号”，而不是靠自然语言赌博。

```python
from pydantic import BaseModel

class ToolError(BaseModel):
    error_type: str   # one of NETWORK, TIMEOUT, RATE_LIMIT, SERVER_ERROR, CLIENT_ERROR, AUTH
    message: str
    retryable: bool
    fallback_used: bool = False
```

工具实际返回时，判断异常类型填好字段。这样 Agent 从响应里读到 `error_type == "TIMEOUT"`，就能根据 prompt 中的指示做出正确决策。

**踩坑点**：别手动拼接 JSON 字符串，用 Pydantic 保证结构，不然 Agent 容易被多余的转义干扰。

### 2. 工具层内置指数退避重试

对可重试错误（网络超时、503、429 等），在工具函数内部用 `tenacity` 实现策略，避免让 Agent 自己乱重试。超时控制必须加上，防止挂死整个调用链。

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import httpx

def is_retryable(e: Exception) -> bool:
    if isinstance(e, httpx.TimeoutException):
        return True
    if isinstance(e, httpx.HTTPStatusError):
        return e.response.status_code in [429, 500, 502, 503]
    return False

@retry(
    retry=retry_if_exception_type(Exception),   # 配合 is_retryable
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=8),
    after=lambda retry_state: log.warning(f"Retrying API, attempt {retry_state.attempt_number}")
)
async def call_external_api(url: str, timeout: float = 5.0):
    # 实际调用
    ...
```

**踩坑点**：429 限流必须根据 `Retry-After` 头来精确等待，别只依赖固定退避，否则可能被对方彻底封杀一段时间。

### 3. 引入熔断器，防止雪崩

当某个外部 API 连续失败超过阈值（如连续 5 次），很可能服务已严重宕机。这时再重试只会加剧 Agent 的失败并增加总延迟。熔断器在这里的价值是：**快速失败 + 保留恢复窗口**。

可以用 `pybreaker` 库，以 API 名称为 key 管理不同服务的断路器。

```python
import pybreaker

weather_breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    timeout_duration=30,     # 30 秒后尝试半开
    name='weather_api'
)

@weather_breaker
def get_weather(city):
    # 内部调用真实 API
    ...
```

一旦断路器打开，工具直接抛出 `CircuitBreakerError`，在外部捕获后返回带有 `fallback_used=True` 的 `ToolError`，并附上缓存数据或预设的降级信息，比如：“天气数据暂时不可用，最近一次查询结果为晴天 22°C”。

**踩坑点**：一定要设置合理的 `timeout_duration`，同时用一个监控钩子记录断路器状态，否则服务恢复了你还一直拿着旧降级数据而不自知。

### 4. 在 Agent 提示词中嵌入错误处理指引

工具能返回高质量错误，并不代表 Agent 会用。需要在系统 prompt 里明确告知 Agent 面对不同类型 `error_type` 时的行为准则，例如：

> - 若 `error_type` 为 TIMEOUT 或 NETWORK 且 `retryable=True`，说明工具已经尝试重试但最终失败。你应该向用户解释网络不稳定，建议稍后重试，**不要**再次调用相同工具。
> - 若 `fallback_used=True`，必须明确指出结果是降级数据，并给出数据时间。
> - 若遇到 RATE_LIMIT，向用户说明当前请求过多，请等待片刻后让用户手动重试。

这样做可以切断 Agent 的“失败-重试-再失败”的恶性循环，同时保证对最终用户透明。

### 5. 限制工具调用轮次，兜底保护

OpenClaw 中可以为 Agent 设定最大工具调用步数。在配置中加一个硬限制，防止错误场景下 Agent 反复调用不同工具直到 token 耗尽。比如 `max_tool_rounds = 8`，足够完成正常多步推理，又不至于死循环。

## 可复用建议清单

- **错误模型统一**：为一个团队的所有 Agent 工具定义一套 `ToolError`，配字段 `retryable` 和 `fallback_used`。
- **超时与重试分离**：工具负责自动重试和超时，Agent 负责语义决策。
- **断路器按服务粒度**：用 API 名称作为断路器键，共享给所有调用该 API 的工具。
- **降级数据必须打标**：缓存数据、静态默认数据带上标识，避免用户误信。
- **监控钩子**：重试次数、熔断状态变化、fallback 使用率都输出到日志，方便后期优化。
- **定期健康检查**：在 Agent 启动或预检阶段探测关键 API，及时触发告警，而不是等用户来踩雷。

## 总结

让 AI Agent 稳定运行的关键，并不是让外部 API 永远不挂，而是当它真的挂了时，整个系统能“体面地”降级，而不是直接崩溃。通过在工具层实现结构化错误、指数退避重试和熔断器，同时在提示词里教会 Agent 如何理解这些错误，我们能将不可控的外部依赖转化为可控的工程边界。

这套模式在 OpenClaw 的工具协作体系中尤其有效——一旦沉淀为模板，团队里其他开发者只要遵循约定，就能立刻获得生产级的错误恢复能力，而不用每次从头造轮子。

---

