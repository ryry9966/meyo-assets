---
title: OpenClaw 子 Agent 不搅乱主会话：session 隔离实践
feedId: 31782
source: 综合讨论
publishedAt: 2026-08-06
---

# OpenClaw 子 Agent 不搅乱主会话：session 隔离实践

## 背景：多 Agent 协作中的上下文污染

在 OpenClaw 的多 Agent 场景里，经常会遇到这样的模式：主 Agent 通过 function calling 调用一个子 Agent 去完成某个复杂子任务，比如搜索、文件处理、代码生成等。一个容易被忽视的问题是——**子 Agent 的“内部思考过程”会原封不动地被拼接到主会话的消息历史中**。

举例来说，主 Agent 说：“帮我查一下这个项目的技术栈”，然后调用了一个子 Agent。子 Agent 内部可能经历了“读取项目文件→分析配置→调用 MCP 工具→再分析→得出结论”的多个步骤，生成了大量的中间思考、工具调用、工具输出。如果所有这些消息都返回给主 Agent，会导致：

- **上下文窗口被撑爆**，主 Agent 很快触及 token 上限
- **信息过载**，主 Agent 被大量无关细节干扰，容易产生幻觉或错误决策
- **计费成本剧增**，因为每一轮主会话都要携带这些冗余内容
- **调试困难**，日志里混在一起，排障时很难分清是哪一层的问题

理想情况应当是：**主 Agent 只拿到子 Agent 的最终结论或结构化摘要，而子 Agent 自身的推理过程留在它自己的 session 里**。

## 问题分析：为什么没有自动隔离

在 OpenClaw 框架中，默认的 Agent 调用逻辑是“对话即上下文”。当一个 Agent 调用另一个 Agent 时，默认行为通常是将被调 Agent 的整个 response（包含其内部 chain-of-thought、tool calls 等）作为工具返回值原封不动地上报。这源于早期的简单串联设计，并未考虑上下文隔离。

有些实现会选择“在子 Agent 内部开启一个新的对话 session”，但如果主会话的 memory、用户画像等信息没有按需传递进去，子 Agent 又会丢失上下文。因此，真正的隔离不是完全切断，而是**有选择地传递输入，有控制地提取输出**。

## 实践做法：在 OpenClaw 中实现 session 隔离

以下基于 OpenClaw 的 Advanced Agent 模式和 session 管理机制，给出一种可落地的方案。不同版本 API 可能略有差异，但核心思想通用。

### 1. 子 Agent 定义为独立 session

创建子 Agent 时，显式指定 `parent_session_id` 为主会话的 ID，但让子 Agent 拥有自己独立的 `session_id`。这样，子 Agent 的所有对话历史都存储于它自己的 session 中，不会污染主 session 的消息列表。

```python
sub_agent = openclaw.create_agent(
    name="researcher",
    instruction="You are a research sub-agent...",
    session_id=openclaw.new_session_id(),  # 独立 session
    parent_session_id=main_session.id,     # 记录父子关系，便于追踪
)
```

### 2. 调用时只获取最终结果

子 Agent 的 response 对象通常包含 `final_answer` 或 `.messages[-1]` 的最终输出。我们可以通过一个薄的 wrapper，在工具函数中只提取最终回答，而不返回中间步骤。

```python
def research_tool(query: str) -> str:
    response = sub_agent.run(query)
    # 只取最后的总结，不返回推理链
    return response.final_answer.strip()
```

如果子 Agent 可能需要多轮交互，可以在工具内设置 `max_turns` 限制，避免失控。

### 3. 利用 structured output 约柬输出格式

要求子 Agent 使用 JSON 结构化输出，这样主 Agent 更容易解析关键信息，同时避免了自然语言中的冗余解释。

```python
sub_agent = openclaw.create_agent(
    ...,
    output_schema={"type": "object", "properties": {"summary": ..., "key_findings": ...}},
)
```

在工具函数中，可以直接将返回的 JSON 对象转成字符串提供给主 Agent。

### 4. 主 Agent 仅传递必要上下文给子 Agent

通过 `run(query, context={"project_info": ...})` 的方式向子 Agent 注入主会话中与该任务相关的关键信息，而不是把整个主对话历史都倒进去。这样可以进一步降低子 Agent 的 token 消耗，同时避免让子 Agent 看到不相干的内容。

## 踩坑点与注意事项

在实践中，以下几个地方容易出问题：

- **子 Agent 异常后的错误信息直接抛出**：如果子 Agent 超时、工具调用失败，默认的异常信息可能包含整个 traceback，再次污染主会话。应在工具函数内部捕获异常，返回一个简短的错误摘要。
- **function call 残留**：有些模型会将子 Agent 中的 function call 信息返回到主会话，如果主 Agent 所用的模型不支持该 function call 格式，会直接报错。务必在 wrapper 中过滤掉所有 `tool_calls`、`function_call` 等消息段。
- **并发调用时的 session 冲突**：如果多个主并发任务共用一个子 Agent session，会互相覆盖对话历史。为每个主任务动态创建独立的子 Agent session 是更安全的选择。
- **token 统计不准**：独立 session 意味着子 Agent 的 token 消耗不会自动归集到主会话的统计中，需要自行记录方便成本核算。

## 可复用建议

1. **封装一个 `SubAgentRunner` 工具类**，内部处理 session 创建、安全参数传递、输出截断、异常标准化，让业务 Agent 只需关注任务。
2. **永远设置 `max_turns`**，避免子 Agent 陷入无限循环或过度探索。
3. **为每个主任务分配独立的子 session**，用完即弃，或设置合理 TTL（例如任务结束后 10 分钟自动过期）。
4. **通过 OpenClaw 的 logging hook**，将子 Agent 的关键事件以结构化方式记入主会话的 meta 数据，而不是消息正文，供调试时参考。
5. **定期清理孤儿子 session**，避免存储资源泄漏。

## 总结

子 Agent 的 session 隔离不是一个复杂的架构，但很多多 Agent 系统在初期都会忽视它，等到上下文爆炸或生产事故后才回头修补。在 OpenClaw 中，通过 **独立 session + 只返回最终答案 + 结构化输出 + 上下文过滤** 的组合，就能用极小的工程代价实现干净的上下文边界。这不仅让主 Agent 更稳定，也让整个链条的可观测性和成本都可控。

后续可以进一步探索异步子 Agent 的通知机制和流式输出截断，但基础隔离做好了，这些高级功能才有一个可靠的土壤。

---

