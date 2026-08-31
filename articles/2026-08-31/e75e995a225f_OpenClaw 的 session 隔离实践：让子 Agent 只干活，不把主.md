---
title: OpenClaw 的 session 隔离实践：让子 Agent 只干活，不把主会话带偏
feedId: 35519
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 里主 Agent 经常要派生子 Agent 去处理搜索、代码生成、数据清洗、批量文件操作等子任务。很多人的第一反应是让子 Agent 直接在主会话里跑，这样省事，但很快就会遇到上下文污染。

默认情况下，如果子 Agent 与主 Agent 共享同一个 session，子 Agent 执行过程中产生的系统提示、工具调用记录、错误重试、中间 JSON、调试日志都会写进同一个 session history。主会话会迅速膨胀，后续推理被大量无关信息干扰。

## 问题

session 污染主要体现在几个方面：

1. **上下文噪音**：子 Agent 为了完成一个任务，可能会经历多次失败的尝试。这些失败过程、重试日志都会留在主会话里。主 Agent 后续推理时，很容易把这些中间状态误认为已经确认的事实。

2. **角色混淆**：子 Agent 的 system prompt、内部指令和主 Agent 的消息混在一起，人类用户和主 Agent 都难以区分哪条消息来自哪个执行单元。

3. **越权读取**：子 Agent 如果直接共享主 session，就能看到主会话之前的短期记忆、工具返回结果，甚至包括一些本不该暴露给子任务的敏感上下文。

4. **调试困难**：当任务出错时，日志里全是一条条没有来源标记的消息，很难定位是主 Agent 决策失误还是子 Agent 执行偏差。

## 做法与步骤

核心思路是：为每个子 Agent 创建独立的 session，只传入完成任务所需的最小上下文，子 Agent 结束后只回传结构化结果，不合并中间日志。

### 1. 创建独立子 session

使用 OpenClaw 的 session 创建接口，分配新的 `session_id`，并通过 `parent_id` 关联主会话。关键是 **不要自动继承主会话历史**，只传递任务描述本身。

```python
child = openclaw.create_session(
    parent_id=main_session.id,
    inherit_context=False,
    ephemeral=True,
    max_tokens=2000,
    tools=["search", "code_exec"],
)
```

### 2. 精简任务输入

不要直接把主会话的整段历史塞给子 Agent。只传递结构化后的任务描述：

```python
task = {
    "goal": "从日志文件中提取错误码出现次数",
    "input_file": "/data/app.log",
    "output_format": "json",
    "constraints": ["只读操作", "不要修改原文件"]
}
result = child.run(json.dumps(task))
```

这样既能保证子 Agent 知道该做什么，又不会看到主会话里的无关讨论。

### 3. 只回传结构化结果

子 Agent 执行完成后，不要返回整个子 session 的日志，而是让其输出一个结构化摘要。可以在子 Agent 的 prompt 里明确要求，或者在代码里做后处理：

```python
summary = result.get("summary")   # 只提取关键结论
main_session.add_tool_result({
    "source": "subagent",
    "task": task["goal"],
    "result": summary,
    "token_usage": child.token_usage
})
```

### 4. 清理子 session

子任务完成后主动关闭或删除子 session，避免 ephemeral session 积累：

```python
child.close()
# 或者
openclaw.delete_session(child.id)
```

### 5. 限制 MCP 工具作用域

如果子 Agent 需要调用 MCP 工具，要显式指定工具白名单，而不是继承主 Agent 的全部工具权限。尤其要避免子 Agent 获得文件写入、数据库修改等能力，除非任务确实需要。

```python
child = openclaw.create_session(
    parent_id=main_session.id,
    tools=["filesystem.read", "search"],
    tool_scope="read_only"
)
```

## 踩坑点

**以为 `parent_id` 就代表隔离**：很多框架里 `parent_id` 只是为了追踪层级关系，并不会自动隔离上下文。必须显式设置 `inherit_context=False` 或同等参数，否则子会话仍然会看到主会话历史。

**返回值没裁剪**：把子 session 的完整日志塞回主会话，等于白隔离。一定要在回传前做摘要或截断。

**MCP 工具共享全局状态**：即使 session 隔离了，如果子 Agent 调用的 MCP 工具直接操作同一个文件系统或数据库，仍然可能污染主 Agent 的工作环境。隔离不仅要管上下文，还要管副作用。

**子 Agent 失控重试**：没有设置 `max_iterations` 或 `max_tokens`，子 Agent 可能在一个错误循环里反复重试，消耗大量资源。务必设置上限。

**并发子任务消息乱序**：如果多个子 Agent 并发执行，却错误地写入了同一个 session，消息会交叉错乱。每个子 Agent 必须有自己独立的 session id。

**清理不彻底**：`ephemeral` 标记不代表立即删除。如果不主动清理，子 session 会积压在存储里，影响后续索引和检索性能。

## 可复用建议

封装一个统一的子 Agent 运行工具，把隔离逻辑固化下来：

```python
def run_isolated_subagent(task_spec, tools, max_tokens=2000):
    child = openclaw.create_session(
        inherit_context=False,
        ephemeral=True,
        max_tokens=max_tokens,
        tools=tools,
    )
    result = child.run(task_spec)
    summary = extract_summary(result)
    child.close()
    return summary
```

这样所有子任务都走同一个隔离入口，避免每个开发者各写一套。同时建议：

- 给子 Agent 的返回结果打上 `source` 标签，标明来自哪个子 Agent、执行了多久、消耗了多少 token。
- 在主 session 里记录子任务元数据，方便审计和排查。
- 定期检查主 session 的 token 增长曲线，如果发现异常增长，优先排查是否有子 Agent 的中间日志泄漏进来。
- 对子 Agent 的权限做最小化授权，默认只读，需要写入时单独审批。

## 总结

session 隔离不是简单地创建一个新 session 就完事。它涉及上下文继承策略、工具作用域、返回结果裁剪、生命周期管理四个层面。只有把这些都处理好，子 Agent 才会成为可靠的工作单元，而不是主会话的噪音源。

在 OpenClaw 的工程实践里，一个比较稳妥的模式是：**主 Agent 负责决策与编排，子 Agent 负责执行与返回摘要，session 边界清晰，权限最小化，用完即删。** 这样既能利用多 Agent 的并行能力，又不会让主会话逐渐退化成一个无法维护的超长上下文。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f2e567b4b37c0e5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a05b04c9fb00ae2c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/088aa0fe11f59787.png)

