---
title: OpenClaw 子 Agent 会话隔离实践：如何防止污染主对话上下文
feedId: 32194
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：多 Agent 协作的“上下文泄漏”问题

在 OpenClaw 的多 Agent 框架里，主 Agent 经常需要把任务委托给子 Agent 完成。默认情况下，子 Agent 会继承主 Agent 的完整对话历史，并且子 Agent 在推理过程中产生的所有中间思考步骤（包括工具调用、反思、重试）都会被写回主 Agent 的上下文。

这带来的问题很明显：

- **Token 浪费**：子 Agent 的内部推理链条对主 Agent 而言是噪音，却要花大量 token 去传递。
- **逻辑干扰**：主 Agent 在后续决策时可能被子 Agent 的中间推理“带偏”，尤其是子 Agent 使用了不同的角色设定或领域知识。
- **调试噩梦**：主对话历史里混杂了子 Agent 的完整思维链，溯源真实意图变得很困难。

工程上我们需要的是一种 **session 隔离**：子 Agent 在独立沙箱里完成推理，只把最终产出的摘要或行动结果回传给主 Agent。

## 问题：默认行为导致的主会话污染

以 OpenClaw 的 `sub_agent.call()` 为例（假设 SDK 类似 `openclaw.agent` 模块），默认调用会把子 Agent 的每一轮回复都追加到主会话的 `messages` 数组中。如果子 Agent 内部需要多步工具调用（例如查询数据库、访问文件、生成代码），整个中间过程都会变成主 Agent 上下文的一部分。

更隐蔽的情况是：子 Agent 在隔离不彻底时，还可能“看见”主 Agent 之前使用过的工具结果，导致重复操作或权限冲突。

因此我们需要一种机制，将子 Agent 的执行完全包在一个独立的 session 里，仅暴露最终响应。

## 做法：利用 `isolated_session` 与结果提取

OpenClaw 从 v0.17 开始为子 Agent 提供了 session 隔离参数。核心思想是：

1. 创建一个子 Agent 专用的空白 session。
2. 只把必要的任务描述和输入参数注入该 session。
3. 执行完成后，丢弃子 Agent 的完整上下文，仅提取最后一条 assistant 消息或结构化输出。
4. 将提取的结果作为工具返回值或普通文本追加到主 session。

实现步骤（伪代码）：

```python
from openclaw import Agent, SessionConfig

main_agent = Agent("main-orchestrator")

# 定义一个工具，内部调用子 Agent 并启用隔离
def call_isolated_sub_agent(task: str, context_data: dict) -> str:
    sub = Agent("data-processor", session_config=SessionConfig(
        isolated=True,            # 不继承任何父 session 历史
        max_tool_calls=10
    ))
    # 显式传入所需上下文，避免依赖主 session
    prompt = f"{task}\n\nContext: {context_data}"
    # 运行子 Agent，返回最终响应
    result = sub.run(prompt)
    # 只取最后一条 assistant 消息作为干净结果
    return result.final_output

main_agent.register_tool("query_data", call_isolated_sub_agent)
```

当主 Agent 调用 `query_data` 时，实际过程是：

- 主会话只看到“调用工具 query_data，传入参数...”
- 子 Agent 的完整推理全部留在临时 session 内，运行结束后该 session 被自动回收。
- 主上下文里只新增一条 `tool` 消息，内容为子 Agent 的最终输出。

这样就实现了“主次分离”。

## 踩坑：一个参数没传对就会漏上下文

在实际落地中，最容易踩的几个点：

### 1. 忘记关闭 `inherit_memory`

`SessionConfig` 的 `isolated=True` 并不一定切断所有记忆载体。如果主 Agent 配置了全局记忆存储（例如 `MemoryStore`），子 Agent 可能默认访问到同一个数据库，导致通过记忆间接污染。需要显式设置 `inherit_memory=False`，或为子 Agent 指定独立的内存命名空间。

### 2. 工具调用在子 Agent 内失败时返回的异常堆栈

当子 Agent 内部工具调用出错，OpenClaw 默认可能会把完整的错误堆栈作为 assistant 消息返回。这些堆栈对主 Agent 是噪音。建议在子 Agent 的 `run` 方法外包裹异常处理，返回用户友好的错误摘要，而不是 raw traceback。

### 3. 子 Agent 需要的权限令主 session 受限

隔离环境下，子 Agent 的工具需要单独授权，因为它们不再能“看到”主 Agent 的工具上下文。如果子 Agent 需要访问与主 Agent 相同的 API key 或文件系统，推荐通过环境变量或密钥注入，而不是从主 session 传递 token。

### 4. 结构化输出丢失

如果你的子 Agent 原本会输出 JSON 并供主 Agent 的后续代码解析，但当启用 `isolated=True` 后，如果只提取纯文本 `final_output`，可能会丢失结构。建议使用 OpenClaw 的 `output_format` 参数强制子 Agent 返回 JSON，然后在工具函数里验证、解析后再返回给主 Agent。

## 可复用建议

**封装为通用的隔离工具工厂。** 写一个 `IsolatedSubAgentTool` 类，统一处理 session 创建、异常捕获、结果清洗，把它注册到所有需要委托的主 Agent 上。这样未来任何新 Agent 要调用子任务，只需实例化该类即可，不用重复处理隔离逻辑。

**始终明确数据契约。** 将子 Agent 的输入和输出定义为强类型模型（如 Pydantic），在主端序列化传入，在工具内部反序列化并喂给子 Agent，再从子 Agent 的返回中解析为同样的模型。类型安全能避免很多因为上下文混入导致的解析错误。

**监控隔离 session 的 token 消耗。** 因为隔离 session 的生命周期短，很容易忽略它带来的成本。最好在 `sub.run` 调用前后记录 token 使用量，并设置 `max_tokens` 硬限制，防止失控。

**不滥用隔离。** 如果子 Agent 的任务必须基于主会话的完整历史才能正确推理（例如“根据我们之前的讨论生成摘要”），那么刻意的隔离反而会打断逻辑链。此时可以只截取最近 N 条相关消息作为上下文传入，而非完全切断。

## 总结

OpenClaw 的子 Agent 会话隔离本质上是一种“沙箱化委托”思维：只传递任务，不传递过程，只回收结论。用 `SessionConfig(isolated=True, inherit_memory=False)` 加上手动的结果提取，能有效保护主会话的纯净度和 token 预算。

在多 Agent 架构中，把每个 Agent 的推理范围约束在最小必要上下文内，是降低 emergent complexity 的关键手法。session 隔离只是起点，进一步还可以结合消息摘要、记忆分层等策略，让 Agent 间的协作更加可控、可预测。希望这篇实践能帮助你避开“上下文爆炸”的坑，构建出更干净的多智能体应用。

---

