---
title: OpenClaw session 隔离实战：让子 Agent 不再污染主会话
feedId: 32185
source: 综合讨论
publishedAt: 2026-08-09
---

# OpenClaw session 隔离实战：让子 Agent 不再污染主会话

## 背景：单会话多 Agent 的隐患

在 OpenClaw 的 Multi-Agent 实践中，我们经常需要让一个「主 Agent」调度多个「子 Agent」去完成特定子任务，比如主 Agent 负责理解用户意图、拆解任务，子 Agent 负责查询数据库、调用插件、执行代码等。很多开发者初期会直接把子 Agent 的对话历史塞进同一个 session 中，或者干脆不关心 session 边界，导致一系列问题：

- **上下文爆炸**：子 Agent 执行过程中产生的中间推理、工具调用结果全部堆进主会话，很快超出 token 限制。
- **语义污染**：主会话中的关键指令或上下文被大量低级交互淹没，模型注意力分散，输出质量下降。
- **错误传递**：子 Agent 的错误尝试、重试逻辑、异常堆栈直接暴露给主 Agent，导致主 Agent 做出错误决策。
- **重放干扰**：在对话式调试时，重建同一个 session 会连带重放所有子任务历史，干扰真实交互。

解决这个问题的核心就是 **session 隔离**：让每个子 Agent 运行在独立的上下文中，只在需要的时候把「干净」的结果返回给主会话。

## 问题定义：子 Agent 与主会话的上下文边界

在 OpenClaw 中，一次 `agent.run()` 或通过 `Orchestrator` 编排的任务，默认情况下不会自动创建独立 session。如果你只是简单地在主 session 的上下文中调用内嵌 agent 的 `run`，并且传递了相同的 `session_id`，那么子 agent 产生的所有消息都会写入同一个会话存储。更隐蔽的是，即使你创建了一个新的 `session_id`，但如果忘记在调用结束后清理或复用，一段时间后内存和存储压力会显著增加。

我们需要达成的目标是：

1. 子 Agent 执行时拥有独立且干净的上下文。
2. 子 Agent 只接收主 Agent 赋予的必要输入，不接触主会话的历史信息。
3. 子 Agent 完成后只向上层返回结构化结果，不暴露过程细节。
4. 会话生命周期可控，用完即弃或缓存复用，避免资源泄漏。

## 实现步骤：手工 session 隔离与封装

OpenClaw 的 session 完全由 `session_id` 标识。最简单的隔离方案就是为每次子任务生成一个唯一的 `session_id`，并在子任务完成后销毁或归档。下面是经过验证的工程化流程：

### 1. 创建独立子 session

```python
import uuid
from openclaw import Agent

def run_sub_task(instruction: str, context: dict) -> dict:
    """在主 Agent 中调用，返回子 Agent 的结构化结果"""
    sub_session_id = f"subtask-{uuid.uuid4().hex[:12]}"
    sub_agent = Agent(
        name="db_query_agent",
        model="gpt-4o-mini",  # 子任务通常模型可以小一些
        system_prompt="你是一个数据库查询助手，只返回 SQL 查询结果和简要解释。",
    )
    # 关键：传入独立 session_id
    response = sub_agent.run(
        instruction,
        session_id=sub_session_id,
        # 不继承主 session
    )
    # 可选：清理 session
    # sub_agent.delete_session(sub_session_id)
    return {"result": response, "session_id": sub_session_id}
```

这种显式生成唯一 ID 的方式简单可靠，适合大多数场景。如果你的 Orchestrator 支持 `context` 注入，可以在子 agent 的 instruction 中直接拼接必要上下文，而不是依赖 session 历史。

### 2. 上下文注入的最佳实践

在不共享 session 的情况下，子 agent 唯一的上下文来源就是你传入的 `instruction` 和 `system_prompt`。因此需要把相关实体、约束、前置结果都编码进一条结构化的指令。例如：

```python
instruction = f"""
查询订单 {order_id} 的物流状态。
当前时间：{current_time}
用户已通过身份验证：{user_verified}
只返回物流状态和预计送达时间，不做任何推测。
"""
```

如果必须传递更复杂的结构化数据（如 JSON），建议固定协议，并在 system prompt 中明确解析规则。

### 3. 错误隔离与优雅降级

子 agent 执行失败（网络超时、工具报错）时，不要直接把堆栈返回给主 agent。可以封装一层 try/except，将异常转化为结构化错误信息：

```python
try:
    result = sub_agent.run(instr, session_id=sid)
except Exception as e:
    result = f"子任务失败: {e}"
```

这样主 agent 看到的永远是一个干净的返回值，不会被内部错误干扰。

## 踩坑记录

**坑 1：忘记设置 `session_id`，串话**

OpenClaw 的 `Agent.run()` 如果不指定 `session_id`，会使用默认会话或上一次引用的 session。有时候你觉得“我创建了新 Agent 实例，应该是隔离的”，实际上底层会复用全局 session 。解决方式是：**总是显式传入 `session_id`**，即使你不关心隔离，至少传一个固定常量，让行为可预期。

**坑 2：session 数量膨胀**

每个子任务都生成一个 session 却不销毁，在高并发场景下会迅速积累大量无用的 session 数据。建议根据任务类型决定生命期：
- 无状态查询：使用完立刻删除 session。
- 需要多轮交互的子任务：保留 session 直到子任务完成（例如子 agent 需要和用户确认信息），然后删除。
- 可缓存的 session（如频繁调用同一类子 agent）：用固定前缀 + 功能标识复用少量 session，但必须在每次调用前手动清空历史消息。

**坑 3：Timing 问题导致主 session “未读”消息残留**

在使用 WS/异步回调时，子 agent 的结果可能以消息事件的形式推送到主 session 的回调中。如果不希望这些事件污染主会话 UI，请在回调中做过滤，或者使用 `session_id` 将事件路由到各自的监听器。

## 可复用建议

1. **封装“无状态子 Agent 执行器”**  
   创建一个工具函数 `run_isolated(agent_config, instruction, context)`，自动生成 `session_id`，执行完毕自动销毁，返回 JSON 结果。可以作为 MCP 工具注册，让主 Agent 直接调用。

2. **统一上下文序列化格式**  
   制定一个标准的子任务描述 schema，包含 `task_type`、`payload`、`constraints`。子 agent 的 system prompt 固定解析该格式，减少 prompt 工程重复工作。

3. **引入轻度上下文压缩**  
   对于那些不得不传递一定历史信息的场景（如递归任务），先用主 agent 对历史做摘要，将摘要作为子 agent 的上下文，而不是传递原始长对话。这样既能提供背景，又不会拖垮窗口。

4. **监控与告警**  
   对 session 创建数、活跃 session 数、平均生命周期做简单埋点，设置阈值告警。防止某个子 agent 不断创建 session 而不释放。

## 总结

Session 隔离是 OpenClaw Multi-Agent 体系下保证稳定性和结果质量的基本功。核心思路就是 **独立 session + 注入必要上下文 + 控制生命周期 + 结构化返回**。它不会增加太多代码，但能显著减少 token 浪费、提高指令遵循度，并让调试变得清晰。早期花一点时间封装好隔离逻辑，后期复杂编排时就能避免大量不可复现的怪问题。

---

