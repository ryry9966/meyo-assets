---
title: OpenClaw Session 隔离实践：子 Agent 只回结论，不把中间过程带回主会话
feedId: 34119
source: 综合讨论
publishedAt: 2026-08-22
---

在用 OpenClaw 做多任务编排时，主会话很容易被子 Agent 拖垮。一个日志分析子任务如果把 tool output、reasoning、重试记录全部 append 到主 session，主 Agent 的注意力会被无关细节稀释；更隐蔽的是 memory 污染——子 Agent 写下的临时事实会进入全局记忆，影响后续任务检索。

这里说的“隔离”，不是把子 Agent 输出藏起来，而是把子任务的上下文、记忆、工具权限限制在独立 session 内，只回传结构化结论。

## 问题拆解

子 Agent 污染主会话通常来自三类边界没有收口：

1. **上下文污染**：子 Agent 的中间消息直接写入主 session 的 message list。主会话 token 迅速膨胀。
2. **记忆污染**：子 Agent 调用记忆工具写入临时结论、实体或偏好，污染全局 memory。
3. **副作用污染**：子 Agent 继承写文件、发消息、改状态等工具权限，通过共享 MCP 连接影响外部系统。

第一类最容易发现，第三类往往在故障后复盘才暴露。

## 做法：给子 Agent 建独立 session

在 OpenClaw 里可以给子 Agent 单独创建 Session，主会话只持有一个引用。示例配置如下，不同版本字段名可能有差异，但思路一致：

```python
sub_session = Session(
    parent_session_id=main.session.id,
    inherit_context=False,          # 不合并主上下文
    context_budget=12000,           # 子任务独立 token 上限
    allowed_tools=["file.read", "mcp.logs.query"],  # 最小权限
    memory_policy="readonly",       # 禁止写主 memory
    return_mode="summary",          # 只回传摘要
)

result = await main.agent.spawn(
    task="分析 /var/log/app/error.log 中的 top 错误码",
    session=sub_session,
    max_steps=12,
    timeout=180,
)
main.session.add_note("log_analysis", result.summary)
```

关键点：

- `inherit_context=False`：子 Agent 不会把主会话历史带进自己的 context。
- `memory_policy="readonly"`：允许读必要的记忆，但禁止写回。
- `return_mode="summary"`：子 Agent 只把最终摘要回传，不影响主会话的消息序列。

子任务失败要做兜底，不要让它把整段 traceback 带回主会话：

```python
try:
    result = await task.run()
except TimeoutError:
    result = {"status": "timeout", "summary": "子任务超时", "artifacts": []}
finally:
    await sub_session.close()
```

## 踩坑点

1. **`inherit_context=False` 不等于记忆隔离。** 上下文和历史记忆往往分开存储。必须显式设置 memory namespace 或只读策略，否则子 Agent 仍可能通过 memory 工具写入全局记忆。

2. **MCP 工具白名单不能完全隔离副作用。** 即使只允许查询类工具，如果 MCP server 本身保持连接状态或临时数据，子 Agent 的查询可能影响主会话后续调用。例如 SQLite MCP 里创建临时表、修改 schema。对高风险 MCP，建议给子 Agent 新建独立 MCP session，或确认服务器支持只读模式。

3. **摘要模式可能丢失关键证据。** 不要只依赖模型总结。要求子 Agent 输出固定 JSON，并在需要时把大段内容写入 artifact 文件，只回传路径和摘要。

4. **子 Agent 的 system prompt 要写清边界。** 否则模型容易“热心”地把中间日志、完整错误码都写进摘要，甚至主动调用记忆工具。可以要求：“不要输出完整日志，不要调用 memory 写工具，只返回 JSON 结论。”

5. **并发子 Agent 不要直接写主 session。** 多个子任务完成后直接 `main.session.add_note` 容易产生竞争。应该收集结果后统一写入。

6. **记得 close 子 session。** 超时或异常时如果子 session 没有关闭，可能残留资源并继续占用上下文。

## 可复用建议

- 封装一个 `spawn_isolated()` 工厂，默认参数设为 `inherit_context=False`、`memory_policy="readonly"`、`return_mode="summary"`、`context_budget=12000`。
- 子 Agent 使用固定输出模板：

  ```json
  {"status": "ok", "summary": "...", "artifacts": ["/tmp/xxx.json"]}
  ```

- 给子 session 加 metadata 标记 `{source: "subagent", task_id: ...}`，方便事后审计。
- 监控主 session 的 token 增长和 memory 新增条目，验证隔离是否生效。发现 `source=subagent` 的 memory 误写，及时清理。
- 子 Agent 上下文预算设小一点，不要让它无上限跑长任务。

## 总结

OpenClaw 的 session 隔离不是把子 Agent 藏起来，而是把上下文、记忆、工具三条边界收口。默认情况下，子 Agent 应只拿最小权限、只写独立 memory namespace、只回传结构化摘要。这样主会话才能保持干净，子 Agent 也能放心跑长任务而不会把噪声带回主流程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/99b7ff956ec909dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b529e9315efdadfb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4df84e71864eaa98.png)

