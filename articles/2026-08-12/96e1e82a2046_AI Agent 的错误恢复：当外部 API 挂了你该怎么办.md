---
title: AI Agent 的错误恢复：当外部 API 挂了你该怎么办
feedId: 32734
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：Agent 不是运行在真空中

很多基于 OpenClaw 搭建的自治 Agent 都会深度依赖外部 API——不管是调用大模型、查数据库、触发 Webhook，还是通过 MCP 工具链访问第三方服务。现实是，这些外部接口随时可能抽风：超时、限流、返回 5xx、回包格式突变，甚至整个服务宕机几十分钟。

这些问题在小规模脚本里可能只是偶尔报错，但在长时间运行的 Agent 任务或生产级自动化流程中，一次未处理的 API 故障就可能导致整个任务链中断、上下文丢失，甚至产生脏数据。所以我们需要的不是“祈祷接口稳定”，而是一套工程化的错误恢复机制。

这篇文章会基于 OpenClaw 的插件体系与 MCP 工具调用，分享一些可落地的错误处理方案，重点解决三个问题：怎么检测故障、怎么优雅降级、怎么自愈恢复。

## 问题拆解：API 故障的真实伤害模型

我们先不急着写代码，先理清常见的故障模式：

1. **瞬时抖动**：偶尔超时或 502，重试能解决。
2. **持续性限流**：429 频繁返回，需退避策略，否则可能被临时封禁。
3. **数据语义错误**：接口返回 200 但内容格式不合预期（比如字段缺失、类型变化），Agent 解析崩溃。
4. **下游完全不可用**：服务宕机，持续失败，无法在短期恢复。

每种模式的伤害不同：瞬时抖动适合重试；持续限流需要动态降频；语义错误需要防御性解析+异常兜底；完全不可用则必须触发降级路径，用缓存、替代数据源或暂停依赖该 API 的子任务，防止 Agent 陷入死循环。

一个无保护的 Agent 在遇到故障时，典型的行为是反复报错、耗尽重试预算、丢失上下文后退出。这显然不能接受。

## 实践做法：在 OpenClaw 中建立三层保护

以下方案基于 OpenClaw MCP 插件体系，适用于自定义工具或通过 MCP 桥接的外部服务。

### 第一层：幂等重试 + 自适应退避

最简单的方案是给所有出站 API 调用包一层带抖动（jitter）的指数退避。在 OpenClaw 插件中，你可以这样封装一个通用请求函数：

```python
import asyncio
import random

async def api_call_with_retry(fn, max_retries=3, base_delay=1):
    last_exc = None
    for attempt in range(max_retries):
        try:
            return await fn()
        except (aiohttp.ClientError, asyncio.TimeoutError) as e:
            last_exc = e
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            await asyncio.sleep(delay)
    raise last_exc
```

但仅此不够。遇到 429 时必须尊重 `Retry-After` 头；如果 API 没有返回该头，则使用更保守的退避策略，并将本次限流事件写入 Agent 的告警日志，方便后续调整调用频率。

**踩坑点**：很多人会对所有异常无差别重试，但 4xx（非 429）通常表示请求本身有问题，重试毫无意义，反而浪费资源。建议只重试 5xx、429 和网络层异常。

### 第二层：防御性解析与 Schema 校验

API 返回 200 但内容结构变化是最危险的失败模式，因为重试机制对它无效。Agent 会拿着错误数据继续执行，后果难以预料。

最佳实践是：**永远不信任 API 回包**。在 MCP 工具函数中，先用 Pydantic 或 JSON Schema 校验关键字段：

```python
from pydantic import BaseModel, ValidationError

class WeatherResponse(BaseModel):
    temperature: float
    description: str

try:
    data = WeatherResponse.model_validate(raw_json)
except ValidationError:
    # 启动降级逻辑，记录告警
    data = get_fallback_weather()
```

如果校验失败，立刻抛出明确的内部错误，并在 Agent 侧捕获这一异常，走降级路径。千万不要用 `dict.get` 层层设默认值来掩盖问题，那会把数据异常扩散到下游。

**踩坑点**：Schema 校验可能带来性能开销，尤其是大体积回包。建议只校验 Agent 实际依赖的关键字段，而不是整个 JSON 结构。

### 第三层：降级策略与熔断器

当某个外部 API 不可用时间超过重试窗口，Agent 必须主动降级，而不是傻等。具体怎么做取决于该 API 在任务中的角色：

- **非核心能力**（例如附加信息查询）：直接跳过，在最终报告中标明“部分数据不可用”。
- **可替换的服务**：切换到备用供应商或本地缓存。例如天气 API 挂了，返回最近一次成功获取的数据，并加时间戳标记。
- **关键依赖**：暂停依赖该 API 的整个子图任务，将任务状态持久化到 OpenClaw 的任务存储，等待外部接口恢复后自动重续。

熔断器模式在这里很有用。用 `circuitbreaker` 库或手动实现状态机：连续失败 N 次后，熔断器打开，拒绝新请求一段时间（例如 30 秒），之后进入半开状态探测恢复情况。这能防止 Agent 不断冲击已经过载的下游服务，也节省自己的重试成本。

示例骨架：

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
async def call_remote_api():
    # 实际请求
    ...
```

当熔断器打开时，直接抛出 `CircuitBreakerError`，在 Agent 上层被捕获，并触发预设的降级路径。

**踩坑点**：如果 Agent 内有多个工具函数共享同一个外部 API，熔断器要作用在统一出口上，否则每个工具各自计数，可能有的熔断有的没熔断，行为不一致。建议在插件内封装一个统一的“API 客户端”层。

## 可复用的建议

1. **错误分类前置**：在 API 客户端层面就把异常分成可重试、不可重试、需降级三类，Error 本身携带分类元信息，便于 Agent 决策。
2. **Agent 侧需要感知“部分失败”状态**：不要总返回单一成功/失败结果。设计一个标准化的工具返回结构，包含 `status`、`data`、`degraded`、`error_reason` 等字段，让 Agent 能根据降级程度调整自己的计划。
3. **记录外部依赖的健康度**：将每次 API 调用的延迟、状态码、失败信息输出到结构化日志，配上简单的健康评分，一旦某服务健康分持续低于阈值，可以触发预警或自动切换。
4. **在开发阶段就注入故障进行测试**：利用 Chaos Monkey 思路，手动改写 MCP 工具返回，模拟超时、格式错误、503，验证 Agent 的恢复行为是否符合预期。这是避免“生产环境第一课”的有效手段。

## 总结

AI Agent 的错误恢复不是写一个 try/except 就完事，它需要在插件设计、工具接口规范、任务调度三个层面同时发力。核心原则是：**让 Agent 对失败有感知能力、有决策余地、有退路可走**。幂等重试应对瞬时故障，Schema 校验防数据污染，熔断+降级应对持续不可用。三者组合在一起，才能让你的 Agent 在外界一片混乱时仍然保持起码的体面。

在 OpenClaw 社区实践过程中，你会发现这套方法不仅适用于外部 API，也同样适用于 MCP 工具、本地插件、甚至与大模型的推理交互。把错误恢复做成基础设施，而不是散落在各个工具里的补丁，是 Agent 从玩具走向可靠生产力的关键一步。

---

