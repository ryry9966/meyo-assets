---
title: OpenClaw 的 session 隔离实践：子 Agent 跑完不把主会话搞脏
feedId: 35448
source: 综合讨论
publishedAt: 2026-08-31
---

# OpenClaw 的 session 隔离实践：子 Agent 跑完不把主会话搞脏

## 背景

在 OpenClaw 里做多 Agent 协作时，子 Agent 经常被用来做搜索、读代码、跑数据、生成中间产物。一个典型的模式是：主 Agent 规划任务 → 派生子 Agent 执行 → 拿回结果继续决策。这个链路本身没问题，但很多人在实现时直接把子 Agent 放进主会话里跑，结果主会话的上下文很快就乱掉了。

本文不讨论该不该用子 Agent，只讨论一个更具体的问题：**子 Agent 的工作过程怎么不污染主会话。**

## 问题：污染从哪里来

默认情况下，如果子 Agent 没有做显式隔离，污染主要来自三个地方：

1. **session 继承**：子 Agent 复用父 session，所有 message、tool_use、tool_result 都写进同一个消息列表。
2. **memory / vector store 共享**：子 Agent 写入的长期记忆或 embedding 结果进了同一个命名空间，主 Agent 后续检索时被带偏。
3. **工具结果回传**：子 Agent 把完整的 tool_use/tool_result 序列返回给父 Agent，父 Agent 当成自己的工具调用去解析。

这三个问题叠加起来，主 Agent 的上下文会迅速膨胀，且充满大量与自己当前规划无关的中间推理。更麻烦的是，主 Agent 可能因为看到子 Agent 的失败堆栈或临时输出，做出不必要的“纠偏”动作。

## 做法：三步切断污染

### 1. 给子 Agent 创建独立 session，并切断 parent 关联

不要省略 `parent_id`。很多框架在省略时会默认继承当前上下文。显式传 `None` 才是真正切断。

```python
from openclaw.session import SessionManager

sm = SessionManager()
child_session = sm.create_session(
    parent_id=None,          # 关键：不要挂到主 session
    scope="subagent:research",
    ttl=600
)
result = await child_agent.run(task, session_id=child_session.id)
sm.close(child_session.id)
```

如果用 YAML 配置，建议显式声明隔离模式：

```yaml
subagent:
  isolation:
    session_mode: independent
    parent_link: false
    ttl_seconds: 600
```

### 2. 结果只回传结构化摘要

不要让子 Agent 把完整消息数组贴回主会话。在子 Agent 结束后，只提取主 Agent 真正需要的信息。

```python
summary = {
    "status": result.status,
    "output": result.final_output[:2000],
    "files_created": result.files_created,
    "error": result.error_type
}
return summary
```

父 Agent 只接收这个 `dict`。这样即使子 Agent 内部跑了几十轮工具调用，主会话里也只多了一条摘要消息。

### 3. 结束即回收，并隔离 memory 写入

如果子 Agent 需要写长期记忆，使用带前缀的 namespace，不要写默认 global。关闭 session 后可以依赖 TTL 让系统自动清理，但最好主动调用 close。

```python
child_agent.memory_namespace = f"subagent:research:{task_id}"
```

## 踩坑点

1. **只隔离 session，不隔离 MCP 连接**。子 Agent 通过同一个 MCP server 调用工具时，server 端的会话状态（游标、临时表、分页位置）可能被复用。需要为子 Agent 单独建立 MCP 连接，或使用只读 MCP。
2. **parent_id 设了 null，但 scope 没设**。排查日志时所有子任务都显示成 anonymous，反而更难定位。建议 scope 唯一，例如 `subagent:research:20250101-xxxx`。
3. **TTL 设置过短**。子 Agent 执行超时被回收，父 Agent 拿到 `None`，可能被当成“任务成功”继续往下走。
4. **并发场景下的命名冲突**。多个子 Agent 同时写同一个 memory key，或同时创建 session 时 scope 撞车。需要加随机后缀或统一由调度器生成 ID。
5. **只关 session 不清 summary**。父 Agent 上下文里留了一堆 `child_session_id` 和中间状态，仍然会让后续规划分心。

## 可复用建议

- **封装一个统一的子任务执行器**：`run_isolated_subagent(agent, task, ttl=300)`，内部处理创建、执行、摘要、关闭、异常回传。不要让每个开发者自己手写隔离逻辑。
- **控制返回体积**：子 Agent 摘要控制在 2KB 以内，大结果写文件，只回传文件路径。
- **写一个隔离性测试**：记录主 session 的 message count，跑完子任务后再记录，断言增量只包含一条 user 消息和一条 assistant 摘要消息。
- **日志里同时打印 parent_session_id 和 child_session_id**，方便追踪调用链和排查污染来源。

## 总结

session 隔离不是简单传个新 id，而是要切断 session 继承、memory 共享、MCP 状态复用三件事。工程上最好收敛到一个统一的子任务执行器里，避免每个开发者各写一套。主会话干净了，多 Agent 协作的上下文才可控，后续规划才不会被一堆中间过程带偏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0a5785ff890cf77d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6517cb598bcda141.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e8b1afb48c3d8e0a.png)

