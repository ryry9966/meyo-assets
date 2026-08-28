---
title: OpenClaw 的 session 隔离：把子 Agent 当函数调，而不是当聊天对象
feedId: 35098
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 里做复杂自动化时，主 Agent 经常拉起子 Agent 去查资料、跑 MCP 工具、生成候选方案。默认情况下，如果子 Agent 直接复用主会话，带来的不是“协作”，而是上下文污染：子 Agent 的多次重试、大段工具返回、中间推理都会进入主历史。主 Agent 后面做决策时，容易把这些过程信息当成已知事实，甚至被带偏。

## 问题不是“开新对话”那么简单

真正的隔离要处理三层：

1. **上下文注入**：子 Agent 是否拿到了主会话的完整历史。
2. **消息写入**：子 Agent 的运行过程是否会回流到主会话。
3. **状态存储**：memory、KV、插件事件是否跨 session 串。

只给子 Agent 新建一个 session id，不说清楚写入和存储边界，最后还是会污染。

## 做法/步骤

### 1. 子任务独立 session，父只持有句柄

给每个子任务分配独立 session id，例如：

```
parent: main-20250611
child: sub-search-20250611-01
```

父 session 只保存子 session_id 和最终结果，不把子 session 的完整 history 合并进来。

### 2. 子 Agent 的输入要结构化，不要给聊天记录

不要传：

```
请根据我之前的分析继续……
```

而是传：

```json
{
  "task": "search_and_summarize",
  "context_snapshot": {
    "target": "OpenClaw MCP tool error",
    "constraints": ["no login required", "max 3 sources"]
  },
  "result_schema": {
    "summary": "string",
    "sources": ["url"],
    "status": "ok|error"
  }
}
```

这样即使子 Agent 需要多轮工具调用，也只在自己的 session 内部进行，不需要父历史参与。

### 3. 工具返回在子 session 内裁剪

MCP 工具很容易返回大 JSON、完整 HTML 或全量日志。如果子 Agent 把原始工具返回直接当结果抛给父，和污染主会话没什么区别。正确做法是：在子 Agent 内部先做一次 summarization，只把满足 result_schema 的字段返回。

```
tool_raw -> child session -> summarize -> parent result
```

父会看到的只有：

```
status: ok
summary: ...
sources: [...]
```

### 4. 用策略强制 result_only，而不是靠 prompt 自觉

在 OpenClaw 的 subagent runner 或自定义插件中，可以加一层策略控制。示意配置如下，实际字段名按你的版本调整：

```yaml
subagent_policy:
  session_scope: isolated
  result_only: true
  max_tool_calls: 8
  token_budget: 6000
  memory_namespace: "sub-{{session_id}}"
  error_mode: compact
```

如果暂时没有策略层，就在子 Agent 的 system prompt 里写死输出格式，并要求“不要输出过程日志，不要输出重试步骤，最终只返回 JSON”。

### 5. 对需要延续的子任务，用续用 session + 摘要回传

有些子任务一次跑不完，需要下轮继续。这时可以在子 session 内保留完整历史，但父 session 只保存一个 `resume_token` 和最后一次的摘要。不要在父历史里放子 session 的中间消息。

## 踩坑点

**全局 memory 写入**

最常见的污染不是聊天记录，而是 memory。子 Agent 如果在执行中调用“记住这个信息”，而这个 memory 没有按 session 命名空间隔离，就会写进主会话记忆。之后父 Agent 每轮都会看到这些只对子任务有意义的信息。

解决：memory 写入必须带 namespace，例如 `memory://sub-session-id/...`，或者禁止子 Agent 主动写 memory。

**MCP 工具状态回流**

有些 MCP server 会返回完整状态对象。工具在父 session 里可读，但一旦被子 Agent 调用，原始返回也容易回流到父。做一层包装函数，只返回 `ok / error / usage`，详细内容只存子 session。

**插件监听 completion 事件**

有些插件会在每个 session 完成后自动总结并发送到主会话。子 Agent 完成后，也可能触发这个插件，导致主会话多出一条“子任务执行摘要”。看起来清晰，但积累多了仍然是噪声。需要把插件的触发范围限制在主 session。

**过度隔离导致反复追问**

如果子 Agent 拿到的背景太薄，它会反复追问或做出错误假设。隔离不是不给上下文，而是给“足量但只读”的上下文快照，并且把约束写清楚。一次给全，比两边来回补历史更省 token。

## 可复用建议

- 把父子和子 Agent 之间的交互定义为“任务对象”，而不是自然语言聊天：

```
输入：{task, context_snapshot, result_schema}
输出：{status, result, error_code, artifacts_ref}
```

- 子任务失败时，只回传 error_code + 简要原因 + 建议动作，不回传完整错误栈。
- 关键工具调用前，在子 Agent 内做参数校验，避免把试错过程带到父层。
- 如果主会话开始变长，先怀疑是不是子 Agent 的中间输出回流，而不是急着压缩上下文。
- OpenClaw 里做插件或 MCP 集成时，所有跨 session 的写操作都显式指定 namespace，默认不写主 session。

## 总结

Session 隔离的核心，不是“另开一个窗口”，而是把子 Agent 的**过程**和**结果**分开。主会话只需要知道“这个子任务做完了没有、结果是什么、下步动作是什么”，不需要知道它中间查了几次、失败了几次、用了什么临时 prompt。

工程上，把子 Agent 当无状态函数调用：输入结构化、输出结构化、过程不可见、状态按命名空间隔离。这样主会话才能保持稳定，不会跑了几轮后变成一锅工具日志和半成品推理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/31e15389f61fe767.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ab5bfee649fc58c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/64b86c288a891b0f.png)

