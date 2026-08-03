---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 31510
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

Agent 的可靠性天花板，现在基本不是模型能力决定的，而是依赖链决定的。一个文生图工具调用的 API 超时、一个天气查询服务的 5xx，就足以让整条自动化任务前功尽弃。OpenClaw 这类 Agent 框架把工具调用包装得很友好，反而容易让人忽略底层 HTTP 故障的真实面貌。

## 问题

外部 API 故障不是"挂了"一种状态。它们是：

- **5xx**：服务端真挂了，短暂重试可能有效
- **4xx**：token 过期、权限被回收，重试一万次也没用
- **429**：限流，等多久都没用，你需要的是冷却
- **超时**：可能是网络问题，也可能是对端在慢死
- **DNS/连接错误**：网络层故障，跟服务本身无关

多数 Agent 的默认行为是"失败就再来一次"。这在短时抖动下有效，但在真实故障中会变成重试风暴——把已经过载的服务打得雪上加霜。

## 做法/步骤

**1. 工具层统一错误协议**

让所有工具返回结构化错误，而不是自由文本。OpenClaw 中可以自定义工具异常，至少包含：

```python
class ToolError(Exception):
    def __init__(self, code: str, retryable: bool, message: str):
        self.code = code          # "API_TIMEOUT" / "AUTH_FAILED" ...
        self.retryable = retryable  # Agent 据此决策
        self.message = message
        super().__init__(message)
```

**2. 按错误类型分流，不盲目重试**

把决策权明确交给 Agent，但给出硬约束：`retryable=True` 时最多重试 3 次，指数退避（1s → 2s → 4s）加随机抖动；`retryable=False` 时直接走降级或终止，不要消耗上下文。

**3. 构建降级链**

每个关键外部依赖都准备一个备胎：

- 天气 API 挂了 → 用之前缓存的最近一次数据，标注 stale
- 搜索 Provider A 超时 → 切 Provider B
- 全部失败 → 明确告诉用户"当前不可用"，而不是编造结果

降级链要在工具描述里写清楚，Agent 才有依据去选。

**4. MCP 层加超时兜底**

不要依赖单一 HTTP client 的默认超时。在 MCP server 或工具包装层统一设置连接/读取双超时，并捕获 `TimeoutError` 转换为 `ToolError("API_TIMEOUT", True)`。这一步能拦截大量"卡死"型故障。

## 踩坑点

- **循环依赖重试**：A 工具调用 B 工具，两边各自重试，叠加出指数级等待，最后把 context window 耗光。建议全链路只允许一层重试。
- **把 4xx 当 5xx 重试**：token 过期后重试三次才报错，白白浪费时间和 token。用 `retryable` 字段一刀切区分。
- **缓存不带时间戳**：Agent 拿到昨天的数据当实时数据用。降级缓存必须显式携带 `fetched_at` 并让模型感知。
- **错误信息过长**：把完整堆栈塞给模型，结果上下文被异常日志占满。工具层应该截断，只给 code、简短 message、重试建议。

## 可复用建议

1. 所有外部调用工具，统一走同一个 `safe_call` 包装函数，内聚超时、重试、降级逻辑
2. Agent 的 system prompt 中写死一条：遇到工具错误先看 `retryable`，不要自己猜测原因
3. 每次 API 失败后，要求 Agent 在最终输出中附带一行决策记录（`retried: 2, fallback: cache, ok`），方便事后排查
4. 对降级数据，工具返回时强制标注 `is_stale: true`，防止模型把它当作实时结果汇报

## 总结

外部 API 一定会挂，这不是概率问题，是时间问题。Agent 的鲁棒性不在于让模型变得更聪明，而在于让错误变得可分类、可决策、可降级。把错误协议、重试上限、降级链和超时兜底这四件事在工具层做扎实，你的自动化任务才能在真实世界里活得久一点。

---

