---
title: OpenClaw 的 session 隔离：子 Agent 只交结果，不把上下文带回主会话
feedId: 34173
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里做复杂自动化时，拆子 Agent 是常见操作：主 Agent 负责规划，子 Agent 去执行搜索、读文件、跑脚本、调用 MCP 工具。但子 Agent 一旦跑起来，中间推理、工具输出、重试日志很容易被带回主会话。表现通常是：主 Agent 越聊越偏，开始复述子任务的细节；上下文快速膨胀，token 成本上涨；甚至主 Agent 的系统提示被大量中间结果稀释，后续决策质量下降。

这个问题不是靠提示词里写“子 Agent 不要污染主会话”就能解决。session 隔离需要从历史、工具状态、连接和返回值四个层面做结构性约束。

## 问题拆解

OpenClaw 中子 Agent 污染主会话，常见来源有四个：

1. **历史透传**：子 Agent 继承了主会话完整 messages，执行完后又把完整 transcript 追加回主会话。
2. **共享 memory/变量**：子 Agent 写了全局 memory 或某个命名空间，主 Agent 后续读到脏数据。
3. **MCP 有状态连接复用**：子 Agent 使用同一个浏览器、数据库或文件系统 MCP 连接，登录态、cursor、临时文件等状态残留。
4. **嵌套 Agent 无边界**：子 Agent 再 spawn 孙 Agent，层层返回后主会话被多层执行日志淹没。

## 做法/步骤

### 1. 给子 Agent 独立 session，不继承主历史

创建子 Agent 时，显式分配新的 `session_id`。只传任务需要的裁剪后输入，不要把主会话全量 messages 传进去。

```python
sub = openclaw.spawn(
    name="researcher",
    session_id=f"sub-{uuid4().hex[:8]}",
    input=task_input,          # 只传任务上下文，不传主历史
    return_mode="summary",     # 关键：只回摘要
    max_turns=10,
    allow_nested=False,
)
result = sub.run(prompt)
```

如果 OpenClaw 版本支持 `parent_session`，把它当作审计/追踪字段，不要当作上下文继承通道。

### 2. 子 Agent 返回结构化结果，而不是完整执行轨迹

主会话只需要知道“结论是什么、文件在哪、有没有出错”。可以要求子 Agent 返回结构化 JSON：

```json
{
  "status": "ok",
  "summary": "已提取 3 个关键段落，写入 /tmp/notes.md",
  "files": ["/tmp/notes.md"],
  "next_steps": [],
  "errors": []
}
```

这样主会话每次只增加一两条 tool result，而不是几十条中间消息。

### 3. 隔离 MCP 工具状态

如果子 Agent 要调用浏览器、数据库等有状态 MCP server，避免复用主 Agent 的连接。实践中可以：

- 为子 Agent 配置独立的 MCP 会话，使用临时 user data 目录或 namespace；
- 只读工具可以共享，有状态工具必须隔离；
- 子任务结束后主动关闭连接、清理临时资源。

例如浏览器类 MCP，如果子 Agent 登录了某个测试账号，主 Agent 后续再操作同一个 browser session，就会带着子 Agent 的登录态跑，非常难排查。

### 4. 主会话只记录输入和输出摘要

在子 Agent 结束后，可以用 `main_session.add_note()` 或等价的 tool result 写入一条摘要。不要直接把子 Agent 的 transcript 注入主历史。这样主 Agent 知道“子任务已完成，结果在 X”，但不会被过程细节干扰。

### 5. 控制嵌套深度

默认禁止子 Agent 再 spawn 孙 Agent。确需嵌套时，每层必须新建 session，并且设置更小的 `max_turns` 和超时。否则递归产生的日志会迅速击穿上下文窗口。

## 踩坑点

- **误用 `parent_session` 做上下文继承**：很多污染来自“为了方便”把父 session 历史传给子 Agent。正确做法是只传裁剪后的 input。
- **子 Agent 返回完整 transcript 以为更透明**：主 Agent 不需要知道子 Agent 每次工具调用的原始输出，尤其是重试和报错堆栈。保留摘要和错误码即可。
- **固定子 session id**：在循环或并发任务里，用固定的 `sub-1`、`sub-2` 会导致并行任务互相覆盖历史。每次 spawn 都用随机 id。
- **忽略清理**：子 Agent 结束不关 MCP 连接、不清理临时文件，污染会在后续任务中延迟暴露。
- **只做历史隔离，不隔离工具状态**：历史很干净，但 MCP 连接里的状态还在，这是最隐蔽的污染源。

## 可复用建议

封装一个 `run_isolated_subagent()` helper，强制以下默认值：

- 随机 `session_id`
- `return_mode="summary"`
- `max_turns` 和超时
- `allow_nested=False`
- 独立的 MCP 配置或临时 namespace
- 返回结构化 JSON

顺手做两件事：日志里带上 `session_id`，方便排障；单元测试里断言主会话 messages 增长数量可控，一次子任务只增加 1-2 条。

## 总结

session 隔离的本质是边界管理：历史边界、工具状态边界、连接边界、返回值边界。不要只靠提示词“不要污染”，而是通过独立 session、结构化返回、MCP 隔离和嵌套控制，让子 Agent 只交结果，不把过程带回主会话。这样主 Agent 能保持决策清晰，上下文成本也可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/88410f1756a543eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/eddd4aafb83a2ca6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/097053333695dd95.png)

