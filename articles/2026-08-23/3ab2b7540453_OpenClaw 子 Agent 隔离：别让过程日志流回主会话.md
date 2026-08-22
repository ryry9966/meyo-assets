---
title: OpenClaw 子 Agent 隔离：别让过程日志流回主会话
feedId: 34266
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里把长任务拆给子 Agent 很常见：抓数据、调用 MCP 工具、批量执行插件、跑 codegen。子 Agent 一旦默认继承主会话上下文，麻烦就开始出现——主 session 里混进大量中间推理、原始工具返回和错误堆栈，多轮之后主 Agent 开始“忘事”“跑偏”。

这里不讨论“要不要用子 Agent”，只聊一个具体问题：怎么让子 Agent 不污染主会话。

## 问题

污染通常来自三个方向：

1. **上下文膨胀**：子 Agent 每一步 tool 输出、MCP 大段 JSON、异常堆栈都带回主 session。
2. **注意力漂移**：主 Agent 看到太多子任务细节，把阶段性失败当成最终失败，或者重复处理已完成动作。
3. **状态串扰**：子 Agent 能读/写主 session memory 或共享 store，A 任务的状态覆盖 B 任务。

核心原则只有一个：**子 Agent 不共享主会话上下文，只通过结构化 envelope 回传结论**。把子 Agent 当成一次函数调用。

## 做法

### 1. 给“任务卡”，不给聊天记录

子 Agent 输入只包含目标、输入、约束、成功标准、输出协议，不要给主会话历史。

示例 task_card：

```json
{
  "goal": "抓取指定 API 的前 100 条记录并生成 CSV",
  "input": { "endpoint": "https://api.example.com/data" },
  "constraints": ["不写主 store", "最多 20 次工具调用"],
  "success": "CSV 已上传 artifacts 且行数大于 0",
  "output_schema": "envelope_v1"
}
```

### 2. 创建独立 session

用独立 session id，关闭父上下文继承。下面用伪代码表示，不同 OpenClaw 版本字段名可能有差异：

```python
sub_session = openclaw.create_session(
    session_id=f"sub_{task_id}_{ts}",
    inherit_parent=False,
    context_policy="task_card_only",
    store="ephemeral",
    max_context_messages=12,
    max_tool_calls=20,
    ttl=600,
)

sub = openclaw.spawn_subagent(task_card, session=sub_session)
result = sub.run()
main_session.append(result.envelope)  # 只追加 envelope
sub.close()
```

关键点是 `inherit_parent=False` 必须显式确认；`store="ephemeral"` 避免写主 memory。

### 3. 只回传 envelope

强制子 Agent 最后输出结构化结果，主 session 只保留 summary 和 artifacts，不保留 details 和日志。

```json
{
  "ok": true,
  "summary": "已生成 CSV，共 87 行",
  "artifacts": ["s3://bucket/out.csv"],
  "errors": [],
  "next_hint": null
}
```

接收端可以再做一层校验：超过固定大小或字段不合规的结果，不进入主 session。

### 4. 工具 / MCP 隔离

子 Agent 只给白名单工具，禁止 `memory.write`、`session.mutate` 这类能力。对 MCP 调用尤其要注意：尽量为子任务建立独立 MCP client/session，或至少显式传递子 session_id，避免 MCP server 把记录回调主会话。

### 5. 清理

在 `finally` 中关闭子 session，TTL 兜底，避免孤儿 session 占用资源。

## 踩坑点

- **看似隔离，实际继承**：很多框架默认继承上下文，只设 session_id 不够。验证方法：子 session 创建后打印 messages 数，应该只包含 task_card。
- **MCP 绑定父 session**：调用 MCP 工具时若不显式传 session，返回结果仍写入主会话。最好在子任务入口创建独立 MCP client。
- **store 仍是共享的**：只切 session 不切 store，子 Agent 写 key 时仍会污染主会话。使用 ephemeral store 或为每个子任务加前缀 `sub:{task_id}:`。
- **回传太“诚实”**：即使 session 隔离，如果子 Agent 把完整工具日志塞进 envelope，主 session 仍被污染。限制 envelope 大小，大结果用 artifacts。
- **忘记清理**：异常时未关闭子 session。用 TTL + finally + 定期清理孤儿 session。

## 可复用建议

- 封装 `run_isolated_task()`，内部统一创建/关闭 session、校验 envelope、清理 store。
- 子 Agent 使用更短上下文和更小模型，主 Agent 保留强推理；不必给子 Agent 主会话历史。
- 对子 Agent 输出做 schema 校验，不合规的不进入主 session。
- 主 session 中只记录“任务 ID + summary + artifact 引用”，方便追踪但不膨胀上下文。
- 监控指标：孤儿 session 数、子 Agent token 消耗、主 session 上下文增长率。

## 总结

Session 隔离不是禁止子 Agent 访问信息，而是控制信息回流。工程上把子 Agent 当“一次函数调用”：传参进去，拿返回值出来，过程留在子 session。这样主会话能保持轻量，子 Agent 也可以放开手做一些脏活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1eed525f45b82beb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/eb6eb1e70ff49d77.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1b1680223516ade3.png)

