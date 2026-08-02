---
title: OpenClaw 的 session 隔离术：让子 Agent 只交结果，不交“历史包袱”
feedId: 31368
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：当 Agent 开始“分身作战”

在 OpenClaw 里引入子 Agent 已经成为常规操作：主 Agent 面对复杂任务时，派一个子 Agent 去搜资料，另一个去查实时数据，自己只负责编排和决策。这种模式下，子 Agent 的调用本质上是把一段封闭的推理过程交给一个独立的上下文去完成。

但很快你会发现一个问题——子 Agent 把它的整个“心路历程”都带回了主会话。不是只交最终结论，而是连同它内部的工具调用记录、多次自我修正，甚至中途的失败重试，统统塞进了主 Agent 的消息列表。主 Agent 的上下文开始无序膨胀，Token 浪费只是表象，更致命的是主 Agent 会误读这些中间步骤，做出错误判断。

这就是 **session 污染**。

## 问题：污染究竟是怎么发生的？

在 OpenClaw 的默认行为里，如果你的子 Agent 是通过 `call_agent()` 或类似机制启动的，子 Agent 的整个 session（Messages 数组）可能会作为工具调用的返回内容，被直接追加到主 session 的消息流中。表面上看，主 Agent 得到了足够详细和丰富的回复，但实际上它拿到的是大量未经筛选的中间噪声。

举个例子：主 Agent 让子 Agent 去总结一篇技术文章。子 Agent 内部可能先调用 `web_fetch` 获取内容，失败一次后切换了备用接口，再调用 `text_summarizer` 生成摘要，最后还自我评价了一下完整性。如果所有步骤都被返回，主 Agent 看到的就不只是一段摘要，还有“HTTP 429，正在重试”这类与它当前决策无关的历史。

这种污染在多次子 Agent 调用叠加后会指数级放大，最终导致：
- 主 Agent 被误导，开始模仿子 Agent 的内部纠错动作；
- 上下文超长，模型性能下降，甚至截断关键信息；
- 调试困难，日志里夹杂大量脏数据。

## 原理：OpenClaw 的 Session 天然支持 Fork

好消息是，OpenClaw 的 session 模型天生就适合解决这个问题。每个 session 都有自己的 `session_id` 和消息列表，而创建子 Agent 时，框架实际上是在 ** 派生出独立的 Session**（可以理解为一个轻量 Fork）。这个子 session 与主 session 之间需要一座可控的桥，而不是直接合并消息。

OpenClaw 的 `SessionContext` 和 `MemoryProvider` 提供了这种能力。关键思路是：让子 Agent 在自己的 session 中执行完整的推理链，但只将 ** 最终状态**（比如一个 `ActionResult` 或一段结构化摘要）注入回主 session，子 session 内部的所有中间步骤、错误重试、工具调用日志都被丢弃。

## 实操：三步构建不污染主会话的子 Agent

假设你已经有一个主 Agent，需要调用一个“知识检索子 Agent”。

**第一步：为子 Agent 创建隔离的 SessionContext**

在 OpenClaw 中，通过 SessionManager 获取一个全新的 session，并显式设置它的 `parent_session_id`，但不开启自动合并。

```python
sub_session = session_manager.create_session(
    parent_session_id=main_session.id,
    auto_merge=False,          # 关键：关闭自动合并历史
    max_rounds=5               # 限制子 Agent 内部交互轮次
)
```

**第二步：在子 session 中运行 Agent，只提取最终输出**

启动子 Agent 时，将 sub_session 作为它的执行环境。子 Agent 完成推理后，通过一个自定义的后处理函数，仅提取 `final_answer` 或 `tool_outputs[-1]`，其余消息全部丢弃。

```python
final_result = run_agent_in_session(
    agent=search_agent,
    session=sub_session,
    user_input=task_prompt,
    post_process=lambda ctx: ctx.messages[-1]["content"]  # 只取最后一条
)
```

**第三步：将最终结果以工具返回值的方式注入主 session**

不要直接把返回内容 append 到消息列表，而是包装成结构化的 `ToolMessage`，让主 Agent 明确知道这是一次子任务的结果。

```python
main_session.add_message(
    Role.TOOL,
    content=final_result,
    tool_call_id=original_tool_call.id,
    name="knowledge_retrieval"
)
```

至此，主 session 里只有干净的“检索结果”，没有任何子 Agent 的内部消息。

## 踩坑点：你以为隔离了，实际上还在漏

在落地这套模式时，有几个隐性问题值得留意：

1. **子 Agent 报错时会泄露**  
   如果子 Agent 内部异常，OpenClaw 默认可能把整个错误堆栈作为回复传回。一定要在 `post_process` 中捕获异常，返回一个格式化后的错误摘要，而不是原始 traceback。

2. **工具调用的副作用**  
   子 Agent 使用的工具（比如写文件、调用外部 API）可能会在主环境中留下状态。如果工具状态是全局共享的，session 隔离也救不了。此时需要用**命名空间**或**事务性回调**限制子 Agent 的副作用范围。

3. **过于激进的截断会让子 Agent 变傻**  
   去掉所有中间过程后，主 Agent 无法了解子 Agent 推理的可信度。可以在结果中附带一个 `confidence` 字段或简短的状态标记（如 `status: succeeded after 2 retries`），既不污染主会话，又保留必要信号。

4. **父 session 上下文不足导致子 Agent 跑偏**  
   子 session 是干净的，它看不到主 session 的历史。所以你必须通过 `user_input` 把必要的上下文（如用户原问题、前置约束）明确传进去，而不能指望它自动继承。

## 可复用的工程化建议

当有多个子 Agent 需要这类隔离时，值得抽象一层：

- **封装为 MCP 工具**：把子 Agent 的调用包装成一个 `SubAgentTool`，内部完成 session 创建、执行和结果过滤。这样其他 Agent 就像调用普通工具一样使用它，无感享受隔离。
- **统一 Session 生命周期管理**：为所有子 session 设置 TTL（存活时间）和最大 token 预算，避免遗忘的子 session 占用资源。可以利用 OpenClaw 的 `SessionCleaner` 定期回收孤儿 session。
- **建立子 Agent 输出契约**：定义返回结构必须包含 `result` 和 `meta` 字段，`meta.status` 可取 `success | partial | failed`，所有中间历史全部留在子 session 中，不纳入契约。

## 总结

Sub-Agent 的 session 隔离不是花活，而是生产级 Multi-Agent 系统的基础卫生措施。OpenClaw 所提供的 Fork 式 session 和灵活的合并策略，让我们可以低成本地实现“只拿结果，不问过程”。只要在设计层面把子 session 当成一次性的异步任务容器，并严防内部噪声外泄，你的主 Agent 就能保持清洁、高效、不会被自己的分身带入坑里。

---

