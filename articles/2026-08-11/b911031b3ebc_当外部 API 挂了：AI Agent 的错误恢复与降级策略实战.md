---
title: 当外部 API 挂了：AI Agent 的错误恢复与降级策略实战
feedId: 32484
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：Agent 的脆弱性不在模型，而在链路

很多跑在 OpenClaw 上的 Agent 已经能稳定完成多步推理、工具调用和插件串联。但一旦某个外部 API 超时、限流甚至直接宕机，整个自动化流水线往往就会中断。更麻烦的是，这类错误不是模型能力问题，而是工程健壮性问题——缺少错误恢复逻辑时，一个 5xx 就能让一个跑了好几分钟的任务前功尽弃。

我用 OpenClaw 搭了一个客户工单自动处理 Agent，工作流大致是：用 MCP 连接内部知识库 -> 调用外部分类 API 确定工单类型 -> 用插件查询库存接口获取备件状态 -> 决定是否自动生成更换单。某天，库存查询 API 因云服务故障挂了近两小时，Agent 直接抛错停下，而当时有 40 多张工单需要处理。事后复盘：不是模型不会处理异常，而是整个流程里根本没有给 Agent “出错后怎么办” 的指令和工具。

这篇文章基于这次事件，整理一套可复用的错误恢复和降级模式，适合把 Agent 跑在生产环境、使用了 OpenClaw + MCP + 外部 API 的同行参考。

## 问题拆解：三种最常见的 API 故障形态

在 Agent 调用链里，我遇到的外部 API 故障大致分三类：

1. **瞬时故障**：网络抖动、短暂超时，重试几次就能恢复。
2. **过载限流**：API 返回 429，或者延迟飙升到不可接受的水平。
3. **彻底不可用**：API 长时间宕机，可能需要小时级修复。

每种情况的处理策略完全不同。如果在代码里只是在 `tool` 调用外部 API 后写一句 `if not response: raise Exception`，这三种故障都会让 Agent 直接终止。而且大多数 MCP 服务端在工具返回异常时，也不会自动做分级处理，错误信息直接丢回给 LLM，LLM 很容易被异常信息带偏，甚至开始编造结果。

## 做法：构建分层的错误恢复机制

我的目标是：不改变 Agent 的主体 prompt 逻辑，把容错能力下沉到工具层。整个实现基于 OpenClaw 的 MCP 服务端框架，对外部 API 调用进行统一封装。

### 第 1 步：定义通用 API 调用包装器

在 MCP 服务端里，不再直接使用 `requests` 或 `httpx`，而是封装一个 `SafeAPICall` 函数，统一处理三种故障：

- **重试策略**：使用指数退避（1s, 2s, 4s, 8s），最多重试 3 次。只对网络错误和 5xx 状态码重试，4xx 一般不重试。
- **限流熔断**：如果连续收到 429 或请求时延超过阈值（我设的是 5 秒），调用一个外部标记文件记录该 API 的“降级状态”，并直接返回预设的降级数据，不再等待超时。
- **缓存兜底**：每次成功的 API 响应写入一个简单的文件缓存（或内存 LRU），key 是请求参数哈希。当 API 彻底不可用时，先尝试读缓存；缓存也没有，返回一个结构化的错误信息，而不是抛出异常。

核心代码结构示例：

```python
import hashlib, json, time
import httpx
from functools import lru_cache

API_TIMEOUT = 10
MAX_RETRIES = 3

async def safe_api_call(url, params=None, fallback=None, cache_ttl=300):
    key = hashlib.md5(json.dumps({"url":url,"params":params}, sort_keys=True).encode()).hexdigest()
    # 尝试读缓存
    cached = read_cache(key, cache_ttl)
    if cached:
        return cached

    last_exc = None
    for attempt in range(MAX_RETRIES):
        try:
            async with httpx.AsyncClient(timeout=API_TIMEOUT) as client:
                resp = await client.get(url, params=params)
            if resp.status_code == 429:
                raise APIRateLimitError("rate limited")
            resp.raise_for_status()
            data = resp.json()
            write_cache(key, data)  # 成功后写缓存
            return data
        except (httpx.TimeoutException, httpx.NetworkError, httpx.HTTPStatusError) as e:
            last_exc = e
            if isinstance(e, httpx.HTTPStatusError) and e.response.status_code >= 500:
                await asyncio.sleep(2 ** attempt)
                continue
            break  # 4xx 或明确的错误不再重试

    # 所有尝试失败，进入降级
    if fallback:
        return fallback
    if cached:
        return cached  # 万一期间有其他进程写了缓存
    # 返回结构化错误，而不是抛异常
    return {"error": "external_api_unavailable", "detail": str(last_exc)}
```

### 第 2 步：让 Agent 理解结构化错误

工具返回 `{"error": "external_api_unavailable", ...}` 后，Agent 如果直接就把它当结果去操作，还是会出问题。所以我在 OpenClaw 的 Agent 指令里加入了一段**错误分支处理说明**，而不是让 LLM 盲猜：

> 如果工具返回的结构中包含 "error" 字段，请根据 error 类型采取行动：
> - `external_api_unavailable`：告知用户当前服务暂时不可用，并根据上下文尝试提供替代信息（如上次已知的缓存数据、建议的操作时间窗口）。
> - `rate_limited`：等待 30 秒后重试，或告知用户稍后再试。
> - `invalid_request`：检查输入参数是否正确。

这条指令让 Agent 把错误当作正常的工作流分支，而不是意外终止。

### 第 3 步：按 API 重要性分级降级

并不是所有 API 失败都需要阻塞流程。我把工单处理中调用的外部接口分了级：

- **关键路径**（影响最终决策）：如库存接口不可用 → 降级策略是生成“待人工确认”的工单，而不是自动创建更换单。
- **非关键路径**（辅助信息）：如客户情绪分析 API 不可用 → 直接跳过，用空字符串填充，不影响主流程。
- **可延迟路径**：如发送通知的 API 失败 → 写入本地队列，后续批量重发。

在 MCP 服务端工具定义时，为每个工具增加一个 `criticality` 参数，由上层决定降级行为。这样不需要在每个 Agent prompt 里硬编码业务逻辑，工具自己就能决定返回什么。

## 踩坑点

1. **缓存过期和脏读**：如果 API 返回的是实时库存，缓存 5 分钟可能让 Agent 做出错误决策。必须在缓存 key 里带上时效要求，并在降级时明确告诉 Agent：“以下数据是 X 分钟前的缓存，仅作参考”。

2. **限流等待把 Agent 超时拖死**：有些 Agent 执行有总超时限制（例如 OpenClaw 的 step timeout），如果重试等待累计太久，Agent 进程本身可能被杀。我的做法是把最大重试总时长限制在 20 秒内，超过就直接降级。

3. **LLM 忽略错误结构，自行发挥**：即使返回了 JSON 错误，模型有时仍会忽略并编造一个“看起来合理”的结果。需要在 Agent 指令中用更强的约束提醒，比如“**绝不要猜测或编造 API 返回结果，当 error 字段存在时必须停止正常处理并报告错误。**” 并做几个 few-shot 示例。

4. **MCP 工具粒度影响恢复逻辑**：如果一个 MCP 工具内部串行调用了 3 个 API，其中第 2 个失败，整个工具返回错误，那 Agent 只能整体重试。更好的做法是把原子 API 调用独立为不同工具，让 Agent 根据错误信息自行编排重试或跳过。

## 可复用建议

- **不要只在模型层思考容错**，把错误拦截在工具层。这样即使换了模型或 prompt，容错逻辑不会丢。
- **错误信息结构化**，让 Agent 能可靠地解析，而不是靠正则匹配错误字符串。
- **给 Agent 一个“知道什么时候放弃”的指令**：比如“若重试失败，生成降级结果并记录日志，然后继续处理下一个工单”。
- **监控工具错误占比**：当某个外部 API 的错误率突然升高，不应该让 Agent 反复降级硬扛，而应触发告警让人介入。我在 OpenClaw 执行过程中添加了一个自定义 hook，在工具返回 `error` 时发送计数到 Prometheus。

## 总结

AI Agent 在生产环境里的真正挑战，往往不是模型能力，而是连接外部系统时的异常处理。把错误恢复逻辑工程化地实现在 MCP 工具层，让 Agent 通过结构化信息感知故障，并按预设分支优雅降级，能够显著降低流程断裂的概率。我落地这套策略后，那次库存 API 宕机事件中，工单处理中断率从之前的几乎 100% 降到了 0——所有工单至少生成了“待人工确认”的结果，没有一条被直接丢弃。

稳定不是靠信任 API 永远健康，而是建立在“假定任何外部依赖都可能失败”的基础上。

---

