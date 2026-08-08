---
title: 在 OpenClaw 中实现 Session 隔离：让子 Agent 不再污染主会话的工程实践
feedId: 32128
source: 综合讨论
publishedAt: 2026-08-08
---

# 在 OpenClaw 中实现 Session 隔离：让子 Agent 不再污染主会话的工程实践

## 一、背景：子 Agent 共享 Session 的隐患

在搭建多 Agent 协作流程（例如主控 Agent + 多个领域子 Agent）时，开发者很容易陷入一个误区：为了省事，让所有 Agent 调用都跑在同一个 Session 里。这在 OpenClaw 这类支持 MCP 工具与持久化会话的框架中，会逐步暴露三个致命问题：

1. **上下文膨胀**：子 Agent 的思考链条、工具调用、中间错误全部写入主 Session，导致主 Agent 很快超出上下文窗口，丢失早期关键指令。
2. **提示词污染**：子 Agent 的系统提示、内部推理可能诱导主 Agent 产生幻觉或偏离目标。
3. **状态耦合**：子 Agent 中途修改的临时变量、会话级缓存可能意外影响主流程，使调试变成灾难。

更隐蔽的是，如果你在 OpenClaw 中启用了记忆模块（如基于向量检索的 recall），这些被“污染”的记忆还可能被持久化，后续对话继续受到影响。

## 二、问题界定：什么才算“污染”

这里先明确一个工程化的判断标准：**主 Agent 的 Session 中，不应该出现任何子 Agent 执行过程的细节，只应该保留其最终产出的结构化结果。**

举个反例：主 Agent 说“帮我查一下用户今年在公司内部的活跃度”，子 Agent 在 Session 里留下了“正在登录内部系统… 查询 MySQL 执行 explain… 发现索引缺失… 重试中… 最终得到 23 条记录”。这些中间步骤对主 Agent 毫无意义，甚至会误导它去讨论不需要的技术细节。

正确的做法是：子 Agent 只返回一个 `{"active_days": 142, "top_project": "OpenClaw"}` 这样的结果摘要，主 Session 里只保留这一条 tool response。

## 三、OpenClaw 中的隔离实现路径

OpenClaw 提供了基于 `SessionManager` 的会话生命周期控制。核心思路是：**为每个子 Agent 调用动态创建独立子 Session，传递最小必要上下文，执行完成后仅将关键结果写回主 Session，并立即销毁子 Session。**

### 步骤 1：封装子 Agent 调用工具

不要直接在主 Agent 的 `tool` 定义中直接调用子 Agent 的逻辑，而是封装一个中间函数，在函数内部控制 Session 的创建与销毁。

假设我们有一个 `internal_search` 子 Agent，定义如下：

```python
# 伪代码，展示关键 API 使用
from openclaw import SessionManager, Agent

async def run_sub_agent(query: str, system_prompt: str, max_turns: int = 10) -> dict:
    # 创建独立子 Session，可以基于主 Session 的 ID 命名，方便追踪
    sub_session = SessionManager.create_session(
        parent_session_id=current_main_session_id,  # 仅用作日志关联，不继承上下文
        inherit_memory=False,   # 不继承主 Session 的记忆
        inherit_context=False   # 不拷贝主 Session 的历史消息
    )
    try:
        sub_agent = Agent(
            session=sub_session,
            system_prompt=system_prompt,
            tools=[...],
            max_turns=max_turns
        )
        raw_output = await sub_agent.run(query)
        # 只提取最终答案，不返回中间消息
        result_text = raw_output.final_answer
        # 必要时做结构化解析，例如要求子 Agent 返回 JSON
        return {"success": True, "data": result_text}
    except Exception as e:
        return {"success": False, "error": str(e)}
    finally:
        # 确保 Session 被关闭，释放资源
        SessionManager.destroy_session(sub_session.id)
```

### 步骤 2：在主 Agent 的 tool 列表中注册

将 `run_sub_agent` 包装成一个 tool 定义：

```python
tool_internal_search = {
    "type": "function",
    "function": {
        "name": "internal_search",
        "description": "Search internal knowledge base with a dedicated sub-agent. Returns a structured result without leaking intermediate steps.",
        "parameters": {...}
    }
}
```

主 Agent 调用此 tool 时，`run_sub_agent` 会透明地完成 Session 隔离，主 Session 只会收到 tool 的返回值（一段文本），没有任何内部推理过程。

### 步骤 3：控制上下文传递量

有时子 Agent 的确需要一些主 Session 里的信息。OpenClaw 支持通过 `session.add_message()` 手动注入必要上下文，而不是继承整个主 Session。我们在创建子 Session 后，只放一条消息：

```python
sub_session.add_message(
    role="system",
    content=f"The user is inquiring about: {query}"
)
sub_session.add_message(
    role="user",
    content=f"Please complete the task: {query}"
)
```

只传入必要的任务描述和约束，避免把主 Agent 的完整对话历史带进去。这样既保证了子 Agent 能完成任务，又不会引发上下文膨胀。

## 四、踩坑点与排障经验

1. **Token 超限的假象**：即使子 Session 独立，如果 `max_turns` 设置过高，子 Agent 在内部仍可能因无限循环而耗尽 token。建议设置 `max_turns` 并配合 `total_token_limit` 参数，超限后自动强制返回已有结果。

2. **并发调用时的 Session 冲突**：如果多个子 Agent 同时运行并共享底层的 LLM 连接池，要注意 OpenClaw 的 Session 对象不是线程安全的。每个子 Agent 必须持有独立的 Session 实例，**不要在多个 asyncio task 间共享 Session 对象**。可以借助 `asyncio.Semaphore` 控制并发数。

3. **子 Agent 错误被吞掉**：因为 tool 返回的是结构化结果，如果子 Agent 返回 `{"success": False}`，主 Agent 很可能无法正确理解错误含义。建议对已知的、可恢复的错误（如超时、部分数据缺失）在 tool 返回值里提供明确的重试提示，例如 `"error_type": "timeout", "suggestion": "retry with shorter query"`。

4. **Session 未销毁导致内存泄漏**：`finally` 块中的 `destroy_session` 非常关键，但还需要处理异常：如果销毁时网络抖动导致会话残留，可以在系统启动时注册一个定期清理过期子 Session 的任务，通过比对 `parent_session_id` 和建立时间自动清理。

## 五、可复用建议：构建一个通用的子 Agent 调用装饰器

为了在多个项目中复用，我抽象了一个 `@isolated_session` 装饰器，将上述逻辑封装：

```python
def isolated_session(max_turns=10, timeout=30):
    def decorator(func):
        async def wrapper(query, system_prompt, **kwargs):
            sub_session = SessionManager.create_session(...)
            try:
                return await asyncio.wait_for(
                    func(sub_session, query, system_prompt, **kwargs),
                    timeout=timeout
                )
            except asyncio.TimeoutError:
                return {"success": False, "error": "timeout"}
            finally:
                SessionManager.destroy_session(sub_session.id)
        return wrapper
    return decorator
```

所有需要隔离的子 Agent 都可以用同一个模式，显著降低心智负担。

## 六、总结

Session 隔离不是银弹，但在多 Agent 系统中是保持主流程清晰、可控的前提。OpenClaw 提供的 SessionManager API 足够灵活，通过如下几个原则即可实现干净的隔离：

- **子 Session 独立创建、最小上下文注入**
- **结果只传结构化摘要**
- **异常与超时统一管理**
- **Session 生命周期和安全清理**

遵循这些实践后，主 Agent 的会话长度稳定可控，调试时也更容易在日志中按 Session ID 区分不同 Agent 的行为，而不是在一大团混合历史里找线索。

---

