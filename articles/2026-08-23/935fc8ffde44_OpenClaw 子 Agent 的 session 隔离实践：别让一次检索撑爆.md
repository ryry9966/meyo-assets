---
title: OpenClaw 子 Agent 的 session 隔离实践：别让一次检索撑爆主上下文
feedId: 34340
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的多 Agent 编排里，子 Agent 经常被用来执行插件调用、MCP 检索、长链路自动化。默认情况下，子 Agent 为了“理解上下文”，会继承父会话的历史消息、工具结果和变量。这在短任务里确实方便，但一旦子 Agent 进入多轮工具调用，返回时就很容易把中间过程一起带回来。

## 问题

主会话被污染有几个典型表现：

- 主 Agent 在后续决策中频繁引用子 Agent 的中间错误输出；
- 上下文窗口被大量工具 JSON、重试记录占满；
- 子 Agent 的调试日志混入主对话，导致后续判断偏移；
- 更隐蔽的是共享 session 状态：子 Agent 修改了某个变量，主 Agent 后续读到的是被覆盖后的值。

这些问题在插件和 MCP 工具链路上尤其明显，因为工具返回值往往又长又不可控。

## 做法/步骤

### 1. 建立隔离边界

主会话只接收结构化返回值。可以在 OpenClaw 配置里开启子 Agent 隔离模式：

```yaml
subagent:
  session:
    mode: isolated
    inherit_context: false
    return_mode: structured
    return_schema:
      status: string
      summary: string
      artifacts: list
```

这样配置后，子 Agent 不再自动继承主会话的完整历史，返回内容也必须符合 schema。

### 2. 显式传递最小上下文

开启隔离后，子 Agent 拿不到主会话上下文，所以要显式传入任务所需的最小输入：

```python
sub = openclaw.spawn(
    task=task,
    context_map={
        "task": task,
        "workspace": workspace,
        "tool_permissions": ["mcp.search", "file.read"]
    },
    session=Session(mode="isolated", parent=None)
)
```

不要传全量历史，只传任务、工作目录和必要权限。

### 3. 让子 Agent 只回传摘要

子 Agent 执行完后，不要直接把完整 transcript 塞回主会话。可以只提取最终结论：

```python
result = sub.run()
main_session.append({
    "role": "tool",
    "name": "subagent",
    "content": json.dumps({
        "status": result.status,
        "summary": result.final_message[:500],
        "artifacts": result.artifacts
    })
})
```

如果框架支持 `return_mode="final_message"`，也要注意子 Agent 最后一条消息可能不是真正的结论，需要配合固定输出格式。

### 4. MCP 中间结果外置

大段检索结果让子 Agent 写入 `workspace/tmp/` 或对象存储，只把路径和行数回传。避免 MCP 工具的原始 JSON 进入主会话。

### 5. 退出前清理

子 Agent 退出前强制调用 `session.compact()` 或 `flush()`，只保留最终答案。如果 OpenClaw 支持 `on_exit` 钩子，可以挂一个清理函数，避免内部 context 残留。

## 踩坑点

- **`inherit_context=false` 后工具权限被拒**：子 Agent 拿不到主会话已确认的工具授权。解决办法是配置 `permission_bridge`，为子 Agent 发一个短期 scope token，而不是直接继承主会话权限。
- **`return_mode="last_message"` 不靠谱**：子 Agent 最后一条消息可能是“我已完成”，但真正的结论在倒数第二条。需要明确要求子 Agent 以固定格式结尾，或在代码里取 `final_answer` 字段。
- **完全隔离导致重复确认**：子 Agent 不知道主 Agent 已经确认过的约束，可能重复问用户。建议在 `context_map` 中放入一份 `constraints` 摘要，而非全量历史。
- **子 Agent 内部 context 爆掉但主会话没被污染**：这种“隐性故障”更难排查。需要给子 Agent 设置 `context_budget` 和 `max_tool_steps`，并在日志中记录子 Agent 的实际 token 用量。
- **全局变量串线**：不要在主会话中用全局变量保存子 Agent 状态，并发的子 Agent 会互相覆盖。

## 可复用建议

- 统一子 Agent 返回协议：`status / summary / artifacts / next_step` 四字段，其他内容一律不进主会话。
- 子 Agent 日志独立落盘，主会话只记录子 Agent 的 `session_id` 和耗时。
- 为所有子 Agent 设置 `max_context_tokens` 和 `max_tool_calls`，防止独立 session 无限膨胀。
- 在自动化测试里断言主会话消息列表中不存在 `subagent_internal` 或 `debug` 类型，防止回归。

## 总结

Session 隔离不是简单的 `inherit_context=false`。真正有效的是“最小上下文输入 + 结构化回传 + 独立日志 + 资源上限”。这样主会话才能保持干净，子 Agent 的失败也不会影响主流程决策。对于 OpenClaw 上的插件和 MCP 实践者，这套边界值得在项目初期就固化下来，而不是等到上下文爆炸后再补救。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ebaa63cb61af70ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/0912c0ab7b5ea478.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e1b02f093b138ac4.png)

