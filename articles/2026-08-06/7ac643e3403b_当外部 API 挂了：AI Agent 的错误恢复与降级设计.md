---
title: 当外部 API 挂了：AI Agent 的错误恢复与降级设计
feedId: 31842
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：Agent 的“软肋”往往在外部

在 OpenClaw、MCP 及其他 Agent 自动化场景中，Agent 的行为高度依赖外部工具：搜索接口、天气预报、翻译服务、知识库查询……你可以把 Agent 想象成一个高级任务编排器，但每次执行 `tool_call` 时，它其实都把一部分可靠性外包给了第三方。

问题来了：这些外部 API 并非总在 SLA 内。网络抖动、限流、凭证过期、服务端 500、响应超时——在生产环境中几乎是日常。任何一个工具调用失败，如果没有设计恢复机制，整个 Agent 任务就会中断甚至进入错误循环。

这篇文章不谈宏大理论，只讲在一线工程实践中，如何让 Agent 在面对外部 API 异常时，做到可控降级、可恢复、可诊断。

## 问题定义：不只是“加个 try-catch”

很多开发者最初的做法是在工具函数外包一层 `try-except`，捕获异常后返回一串错误文本让 LLM 自行消化。这在简单场景下勉强可用，但一旦工具调用链变长，就会暴露出几个痛点：

- **没有重试策略**：偶发超时直接失败，任务被丢弃。
- **错误信息非结构化**：LLM 难以精确判断是“暂时不可用”还是“权限不足”，导致后续决策混乱。
- **无熔断保护**：某个 API 持续失败，Agent 仍不断重试，耗尽自身配额甚至被服务商封禁。
- **缺乏全局降级路径**：当关键依赖不可用时，Agent 没有“备用工具”或“人工兜底”的选项。

真正的恢复机制需要覆盖：重试、熔断、降级、错误传播与监控，每一步都要适配 Agent 的上下文与调用协议。

## 工程做法：五层防御

我们以一个典型的 OpenClaw 工具调用链为例：Agent 需要先调用搜索 API 获取新闻，再调用摘要 API 生成简报。使用 Python 实现，但思路不限于语言。

### 1. 超时与指数退避重试

对所有出站 HTTP 请求强制设置连接超时和读取超时（例如 `connect=2s, read=8s`）。重试使用指数退避，同时引入“jitter”避免惊群效应：

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=8),
    reraise=True,
)
def call_search_api(query):
    ...
```

关键点：**只在瞬态错误上重试**（5xx、网络超时），不要在 4xx 客户端错误上重试（例如 401 表示凭证问题，重试无意义）。否则容易导致服务端把 IP 列入黑名单。

### 2. 熔断器（Circuit Breaker）

当某个工具在窗口内连续失败达到阈值时，直接短路后续请求，避免连锁故障。使用 `pybreaker` 库或自己做一个简单的状态机：

```python
breaker = CircuitBreaker(
    fail_max=5,
    timeout_duration=60,  # 熔断后 60 秒再尝试恢复
    exclude=[ValueError]  # 不触发熔断的异常
)
```

Agent 调用工具时先通过熔断器，若电路断开，直接返回预定义的 `{"error": "tool_unavailable", "reason": "circuit_open"}` 而不发起实际请求。这样 Agent 可以立刻感知并选择备用路径。

### 3. 降级策略：从“返回空”到“功能替代”

搜索 API 不可用时，可以降级为返回缓存的热点新闻或空列表；翻译服务不可用，可以让 Agent 返回原始语言内容并附加说明。关键在于提前为每个工具定义降级响应（Fallback Response），格式要可被 LLM 理解：

```json
{
  "status": "degraded",
  "fallback": true,
  "data": [],
  "message": "Search service temporarily unavailable. Showing recent cached topics."
}
```

如果任务有替代工具，可以让 Agent 在决策时就携带备选方案。例如 MCP 协议中，工具描述可以声明 `fallback_tool`，由客户端在调用失败后自动切换。

### 4. 结构化错误传播

Agent 的错误处理能力受限于错误信息的清晰度。在工具返回的错误中，建议包含：

- **错误码**：`TIMEOUT`, `UPSTREAM_503`, `RATE_LIMITED`, `INVALID_AUTH`
- **建议动作**：`retry`, `skip`, `notify_user`, `switch_tool`
- **可读消息**：简短说明原因，用于 LLM 对话上下文

示例：
```json
{
  "error": "UPSTREAM_503",
  "action": "notify_user",
  "detail": "Weather API return 503; task cannot continue without weather data. Please inform user and abort current step."
}
```

这样 Agent 不必猜测错误性质，可以直接执行建议动作。

### 5. 监控与告警

所有工具调用的成败、延迟、熔断状态都应埋点。可以用 Prometheus + Grafana 或简单的日志聚合。当某个工具的失败率超过阈值时，运维同学能先于用户发现问题。

特别注意：**记录降级路径的触发次数**，用于评估外部依赖的健康度和降级策略是否合理。

## 踩坑实录

1. **无差别重试导致 API 配额耗尽**  
   曾有一个翻译工具返回 403，原因是月度配额已用完，但重试机制仍不断请求，导致 IP 被临时封禁。教训：`4xx` 错误绝对不要重试，应该在错误码表中明确排除。

2. **超时配置未覆盖所有出口**  
   Agent 框架内部有时会维护长连接池，默认超时可能无穷大。一个工具调用卡住几分钟，整个 Agent 线程池被耗尽。每个 HTTP 客户端都要显式设置 `timeout`。

3. **熔断恢复后的“雪崩”**  
   如果熔断恢复时刻大量积压请求同时涌向刚恢复的 API，很容易再次触发熔断。可使用“半开状态”仅允许少量请求探测，成功后再完全恢复。

4. **错误消息里的技术细节淹没有效信息**  
   直接把 `requests.exceptions.ReadTimeout: HTTPSConnectionPool...` 抛给 LLM，不仅浪费 Token，还容易导致 Agent 产生幻觉。必须进行简洁封装。

## 可复用建议

- 封装一个 `SafeToolCaller`，统一处理重试、熔断、降级，让工具开发者只关心正常逻辑。
- 在 OpenClaw 的工具描述中加入 `error_handling` 字段，注明该工具的降级能力和恢复动作，方便 Agent 决策。
- 利用 MCP 的 `Result` 结构中的 `meta` 字段传递结构化错误，不要依赖纯文本。
- 将降级逻辑和业务逻辑解耦，通过配置文件或注册机制管理每个工具的 Fallback。

## 总结

外部 API 挂掉不是偶发事件，而是常态化背景噪声。一个生产级的 AI Agent 需要在依赖方不可用时，仍能以可预期的方式完成任务或暂停等待人工介入。重试、熔断、降级、结构化错误与监控，这五层防御缺一不可。

对于 OpenClaw 社区的用户来说，现在就可以检查你的工具调用链：是不是每个 HTTP 请求都设置了 timeout？有重试判断吗？有熔断吗？如果做不到全部，至少把手动处理的异常改成结构化的 `tool_error` 返回，这能让你的 Agent 从“一碰就倒”变得“皮实抗造”。

---

