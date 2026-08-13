---
title: OpenClaw 实战：子 Agent 的 session 隔离，避免上下文污染
feedId: 32937
source: 综合讨论
publishedAt: 2026-08-13
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

在 OpenClaw 里做复杂任务编排时，经常会把一个大任务拆给多个子 Agent 并行或串行执行，例如让一个 Agent 先搜索资料，另一个 Agent 整理代码。很多同学一开始会直接复用主会话的 context，让子 Agent 把每一步推理、工具调用结果都写回主历史。这样做的后果是：主会话很快被大量中间日志淹没，模型开始混淆上下文，后续主 Agent 的回答质量明显下降，token 成本也直线上升。

问题不在于子 Agent 本身，而在于 **session 没有隔离**。

## 问题表现

主会话被“污染”通常有几种典型症状：

- 主历史里塞满子 Agent 的中间输出，例如一大段工具返回的 JSON；
- 子 Agent 的 system prompt 或角色设定和主 Agent 冲突；
- 主 Agent 在后续步骤中开始引用子任务里的无关细节，甚至把子 Agent 的输出当成用户输入；
- 上下文窗口被快速占满，不得不提前截断或压缩，丢失关键信息。

简单说：**子 Agent 的过程数据不该进入主会话，主会话只需要最终结论。**

## 做法与步骤

以一个常见的 OpenClaw 场景为例，主 Agent 需要调用一个 planner 子 Agent 生成执行计划。推荐的做法如下：

### 1. 为子 Agent 创建独立 session

不要让子 Agent 直接写入 `main_session.id`。单独创建一个有短 TTL 的子 session，并挂上 `parent_id` 方便追踪：

```python
child_session = openclaw.create_session(
    parent_id=main_session.id,
    ttl=300,                # 5 分钟自动回收
    inherit_memory=False     # 关键：不继承主会话记忆
)
```

### 2. 只注入必要上下文

子 Agent 不需要看到主会话的完整历史。把主上下文压缩成一小段“任务卡”，只携带目标、输入参数和必要文档片段：

```python
task_brief = build_task_brief(
    goal="生成 Q3 发布计划",
    constraints=main_context.constraints,
    context_docs=retrieve_docs(query, top_k=3)
)

result = openclaw.run_agent(
    agent="planner",
    prompt=task_brief,
    session_id=child_session.id,
    inject_context=False
)
```

### 3. 结构化结果回传

子 Agent 执行完成后，只提取最终结论写回主会话。不要直接追加 `result.raw_history`：

```python
final_payload = extract_json(result.output)  # 例如 {"status": "ok", "plan": [...]}
main_session.append_message(
    role="tool",
    content=f"[subtask: planner] done. {len(final_payload['plan'])} steps generated."
)
```

### 4. 工具权限做白名单

子 session 里不要复制主会话的全部工具。通过 MCP 或插件层做 namespace 过滤，只给只读或必要工具：

```python
child_tools = filter_tools(
    main_session.tools,
    allow=["read_file", "search", "get_issue"],
    deny_write=True
)
```

### 5. 清理 session

无论在正常结束还是异常路径，都要确保子 session 被关闭或至少触发 TTL 回收：

```python
try:
    result = openclaw.run_agent(...)
finally:
    openclaw.close_session(child_session.id)
```

## 踩坑点

实际落地时，下面几个坑比较常见：

- **用 `fork_session` 但没有 detach**：子 session 仍然和主记忆共享，隔离形同虚设。确认退出时明确 `detach_memory()`。
- **回传了 `raw_history`**：里面包含子 Agent 的所有中间输出，等于把污染又带回来。只传 `final_output`，或者更好的是只传结构化摘要。
- **工具权限继承过多**：直接复制主 session 工具列表，导致子 Agent 可以触发写操作或修改主会话绑定的资源。务必走白名单。
- **异常路径没清理**：子 Agent 跑挂后没有在 `finally` 里回收 session，留下孤儿 session 占用内存和并发额度。
- **注入上下文过多**：为了“保险”把主历史截断后仍塞入大量内容，结果导致子 session 上下文也膨胀，和隔离目标冲突。
- **并发下 session id 冲突**：如果手动拼 `child_{main_id}_{task_name}`，并发时容易重名。建议使用 `create_session()` 返回的唯一 id，不要沿用主 id。

## 可复用建议

如果你的团队需要频繁使用子 Agent，建议封装一个统一的 helper，而不是每次手写隔离逻辑：

```python
def run_isolated_child(agent, prompt, context_brief, tools, parent_id):
    sess = openclaw.create_session(parent_id=parent_id, ttl=300)
    try:
        result = openclaw.run_agent(
            agent=agent,
            prompt=prompt,
            session_id=sess.id,
            inject_context=False,
            tools=tools
        )
        return extract_structured_result(result.output)
    finally:
        openclaw.close_session(sess.id)
```

同时约定回传 JSON schema，例如：

```json
{
  "status": "ok" | "error",
  "answer": "最终答案或摘要",
  "artifacts": ["file_id1", "file_id2"]
}
```

主会话只记录子任务的摘要和产物引用，而不是完整输出。这样即使并发跑多个子 Agent，主上下文也能保持干净。

## 总结

OpenClaw 的 session 隔离不是简单“新建一个会话”这么容易。它围绕三件事展开：

1. **上下文隔离**：子 Agent 不读主历史，只接收精简任务卡；
2. **工具隔离**：子 Agent 只能使用白名单工具，避免越权；
3. **结果隔离**：主会话只回传结构化最终结论，不接受过程数据。

把这三件事固化到 helper 和 review 规范里，子 Agent 才能真正成为安全、可扩展的执行单元，而不是主会话的“上下文污染源”。

---

