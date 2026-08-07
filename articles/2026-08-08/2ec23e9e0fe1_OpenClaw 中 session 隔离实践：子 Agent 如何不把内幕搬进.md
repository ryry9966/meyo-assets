---
title: OpenClaw 中 session 隔离实践：子 Agent 如何不把内幕搬进主会话
feedId: 32053
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景：从一次会话膨胀说起

在 OpenClaw 中编排多 Agent 任务时，我们经常会把可复用的逻辑封装成子 Agent，然后由主 Agent 通过 MCP 工具（如 `call_agent` 或 `task`）进行调用。早期大家的直觉是：子 Agent 干完活，把结果扔回给主 Agent，主会话继续往下走。但很快就发现，主会话的消息列表里莫名其妙出现大量子 Agent 的思考链、工具调用细节甚至内部调试信息，上下文窗口迅速膨胀，token 消耗翻了几倍不止。

更麻烦的是，这些不属于主线程的杂讯会干扰模型判断——主 Agent 看到子 Agent 的中间推理后，可能产生“继承误读”，用子任务的局部结论覆盖整体决策。这就是我们说的**子 Agent 污染主会话**。

根本原因在于 OpenClaw 默认的串行 agent 执行模型：当主 Agent 调用子 Agent 时，默认逻辑是把子 Agent 的完整消息历史（包括 system prompt、user message、所有 assistant 回复和 tool 调用）一股脑拼接到主会话中。对于步骤短、输出小的子任务还能忍受，一旦涉及多轮检索或复杂规划，主会话立刻被污染。

## 问题拆解

我们需要隔离的不是功能调用本身，而是**子 Agent 的上下文空间**。具体表现：

1. 主会话中出现子 Agent 的 `think` 块、中间 tool 结果等非必要信息。
2. 主 Agent 在后续决策时引用子 Agent 的内部提示词或错误输出，产生幻觉。
3. 因为主窗口被占满，真正重要的历史对话被截断或压缩，长程任务稳定性骤降。
4. 敏感数据（如子 Agent 访问的数据库查询语句）泄露到主会话日志中。

理想的隔离效果是：子 Agent 独立持有自己的 session，主 Agent 只拿到一个干净的结构化返回值，看不到内部推理过程。

## 做法：三行配置实现隔离

通过对 OpenClaw 的 `AgentSession` 和 `call_agent` 工具的参数调优，可以组合出一套可靠的隔离方案。

### 1. 为子 Agent 指定独立 session

在调用子 Agent 时，通过 `new_session` 或 `session_scope` 参数强制创建独立会话。以 MCP 工具为例：

```json
{
  "tool": "call_agent",
  "parameters": {
    "agent_name": "sql_reviewer",
    "input": "review the query plan for slow log #1024",
    "session_scope": "isolated",
    "response_mode": "final_only"
  }
}
```

`session_scope=isolated` 会告知运行时：**不要复用当前主会话的上下文，为子 Agent 另启一个干净的 session**。如果你担心会话 id 冲突，可以显式传入 `session_id: "sub_<uuid>"` 以确保唯一性。

### 2. 仅返回最终结果

`response_mode` 是关键。可选值通常包括：
- `full`（默认）：子 Agent 整个会话历史全部退回给主 Agent。
- `final_only`：只返回最后一条 assistant 消息，不返回中间步骤。
- `custom`：结合 `output_mapping` 筛选字段。

多数场景直接用 `final_only` 就够。如果你的子 Agent 需要返回多段结构化数据，可以启用 `output_mapping` 让它只抽出 JSON 块：

```yaml
# 在子 Agent 的配置中定义
output:
  mode: custom
  include_keys:
    - final_answer
    - execution_time
  exclude_patterns:
    - ".*token_count.*"
```

这样返回给主会话的只有精简后的 payload，不再夹带冗长的思考链。

### 3. 子 Agent 内部的 prompt 约束

在子 Agent 的 system prompt 中明确指示“你的输出将直接返回给调用者，请省略推理过程，仅给出最终结论”。示例：

> You are a database index advisor. You MUST output only the final recommendation in valid JSON. Do not include chain-of-thought or tool call summaries.

这能在源头减少子 Agent 产生冗余内容的动机。

### 4. 日志与监控层面隔离

即使会话隔离到位，运行时的日志流仍然可能把子 Agent 消息打印到主 Agent 的 tracking 里。建议在主 Agent 的任务配置中开启 `log_filter_policy: subagent_only`，将子 Agent 的执行日志归入独立文件，并在主会话的 UI 中隐藏。

## 踩坑与排障

**坑 1：子 Agent 依赖主会话上下文。**

有些业务逻辑需要子 Agent 看到主 Agent 之前的对话（例如带上下文的情感分析）。完全隔离会导致子 Agent 收到孤立的 `input`，无法完成任务。此时不能一刀切，需要构建**显式的上下文传递机制**：在主 Agent 调用子 Agent 前，手动拼装一个 `context` 字段，把必要的背景摘要、用户意图等打包进去。

```json
{
  "tool": "call_agent",
  "parameters": {
    "agent_name": "sentiment_analyzer",
    "input": {
      "text": "{{latest_user_message}}",
      "summary_of_previous_turns": "{{history_summary}}"
    },
    "session_scope": "isolated",
    "response_mode": "final_only"
  }
}
```

**坑 2：session 清理不及时导致内存泄露。**

每个独立 session 都占用上下文缓存。一次复杂任务可能产生十几个子 session，如果不主动回收，长期运行的服务会出现内存堆积。最佳实践是给子 session 设置 TTL（time-to-live，存活时间），任务结束后立刻调用 `close_session`。

```python
# 伪代码示例
sub_session_id = f"sub_{uuid4()}"
result = await call_agent(
    agent_name="..., 
    session_id=sub_session_id, 
    ttl=300
)
# 主动关闭
await runtime.close_session(sub_session_id)
```

**坑 3：嵌套调用导致追踪断裂。**

当子 Agent 再次调用孙 Agent 时，默认的隔离策略只在第一层生效。需要明确开启“递归隔离”，即所有子调用均继承隔离标志。检查 OpenClaw 的运行时配置中是否有 `propagate_session_scope: true`，保证整棵调用树都被隔离。

## 可复用建议

1. **封装调用函数**：将所有 `call_agent` 调用收敛到统一的 `invoke_sub_agent(name, input, context=None)` 工具里，内建隔离、TTL、output_mapping 等默认值，避免每次手写。
2. **建立 session 模板**：对高频子 Agent 预定义 `session_template`，包含隔离策略、日志级别、响应模式，部署时直接引用。
3. **监控 token 消耗**：在隔离配置中打开 `return_token_stats: true`（仅返回统计），方便监控每个子调用占用的资源，当某子 Agent 的 token 数异常增高时能够快速定位。
4. **定时巡检**：每周检查主任务的完整消息日志，过滤出 `sub_agent_sessions` 事件，确保没有意外拼入主会话的数据块。

## 总结

OpenClaw 的 session 隔离看似是小配置，实际决定了 Agent 编排能否从 demo 走向生产。核心就三点：**独立会话空间、精简结果返回、明确上下文边界**。把脏活留给子 Agent 内部，把干净的结论还给主 Agent，主会话才能聚焦核心任务，避免上下文被无关信息稀释。

下次再遇到主 Agent 莫名其妙引用 SQL 报错信息时，不妨先查一下子 Agent 的 session scope——大概率是它把整本日记塞给了你。

---

