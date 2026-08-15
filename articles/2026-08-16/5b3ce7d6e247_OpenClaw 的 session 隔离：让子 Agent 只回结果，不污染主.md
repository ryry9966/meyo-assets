---
title: OpenClaw 的 session 隔离：让子 Agent 只回结果，不污染主会话
feedId: 33347
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

在 OpenClaw 里跑多步自动化时，主 Agent 经常需要把任务拆给子 Agent：搜索资料、生成代码、读 MCP 工具返回、做长文档摘要。最初我把子 Agent 的完整输出直接拼回主 session，结果很快遇到问题：上下文快速膨胀，主 Agent 开始“跑偏”，调试时也分不清哪句话是主 Agent 说的，哪句是子 Agent 的中间推理。

后来我把子 Agent 的 session 真正隔离出来，只回传结构化结果，情况才稳定下来。这篇文章记录一下做法和踩坑点。

## 问题

子 Agent 污染主会话通常不是“有没有写进 history”这么简单，而是三类问题叠加：

1. **上下文膨胀**：子 Agent 的中间推理、工具调用、重试过程全部进入主 session，很快撞上 token 上限。
2. **意图漂移**：主 Agent 看到子 Agent 的中间结论后，容易把子任务里的错误假设当成既定事实，后续决策被带偏。
3. **调试困难**：主 session 里混着多轮子任务记录，很难定位是哪个子 Agent 引入了错误。

真正要解决的是：**子 Agent 的过程可以丢弃，但结果必须可信、可回传、可追溯。**

## 做法 / 步骤

### 1. 给子 Agent 独立 session，不要复用主 session

在 OpenClaw 的 agent 配置里，我给子任务单独开一个 session。不要在主会话里直接调用子 Agent 的执行函数，而是通过 spawn/run 的方式启动独立上下文。

```yaml
# openclaw.yaml 片段
subagent:
  researcher:
    session:
      isolate: true
      ttl: 600
    return: final_only
```

关键是 `isolate: true`，它让子 Agent 的消息循环、工具调用、错误重试都发生在自己的 session 里，不会写回主 session。

### 2. 只回传结构化结果，不要回传中间消息

主 Agent 需要的是子 Agent 的最终结论，而不是过程。我会让子 Agent 返回一个固定结构：

```ts
// 伪代码，字段名以你的 OpenClaw 版本为准
const sub = await openclaw.spawn({
  name: "researcher",
  session: { isolate: true },
  prompt: "查一下 OpenClaw 的 MCP 配置最佳实践",
  maxTokens: 4000,
});

const result = await sub.run();
// 只取 finalAnswer，不回传 sub.history
await mainSession.reply({
  role: "tool",
  content: JSON.stringify({
    source: "researcher",
    summary: result.finalAnswer,
    refs: result.refs || [],
  }),
});
```

这里有两个工程细节：

- `sub.history` 不要直接合并到 `mainSession.history`。
- 返回内容尽量用结构化对象，主 Agent 后续解析时更稳定。

### 3. MCP 工具状态也要隔离

只隔离消息还不够。如果子 Agent 调用了同一个 MCP server，而 MCP server 有共享状态，比如写文件、改环境变量、操作同一个数据库连接，那么子 Agent 的副作用还是会“泄漏”到主会话。

我的做法是：

- 给子 Agent 使用只读 MCP 工具，或者
- 使用独立的命名空间/工作目录，比如子 Agent 的 `WORKSPACE` 指向临时目录。
- 如果 MCP server 本身无状态，那问题不大；如果有状态，必须在子 Agent 生命周期结束前清理或隔离。

### 4. 错误处理只回传摘要

子 Agent 失败的堆栈信息如果直接塞给主 Agent，主 Agent 往往会尝试“修复”一个已经过期的子任务。更好的做法是 catch 后只回传错误摘要：

```ts
try {
  const result = await sub.run();
  return { ok: true, summary: result.finalAnswer };
} catch (err) {
  return {
    ok: false,
    error: err.message, // 不要返回完整 stack
    retryable: true,
  };
}
```

主 Agent 只需要知道“失败原因 + 是否可重试”，不需要知道子 Agent 执行到第几轮、哪个工具报错。

## 踩坑点

1. **只隔离了消息，没隔离工具副作用**  
   这是最隐蔽的坑。子 Agent 调用了写文件工具，主会话后续读到了脏数据。建议给子 Agent 配独立工作目录，或者用只读工具。

2. **用同一个 sessionId**  
   如果子任务复用主 session 的 ID，`isolate: true` 会失效，消息还是会写回主会话。检查配置里是否动态生成 sessionId。

3. **子 Agent 输出过长，主 Agent 被“淹没”**  
   即使只回传 finalAnswer，但 finalAnswer 可能很长。回传前做截断或分段，比如只取前 2000 字符，或者让主 Agent 按需再查询子 session。

4. **错误堆栈直接回传**  
   主 Agent 看到大段堆栈后，会尝试理解子 Agent 的实现细节，反而偏离主任务。只回传可读错误信息。

5. **子 Agent 生命周期没设置 TTL**  
   如果子 Agent 进入死循环或长时间无响应，没有 TTL 会把主会话也拖死。给子 Agent 设置超时和最大 token。

## 可复用建议

- **把子 Agent 当成纯函数**：输入 task，输出 result 对象。中间过程不进入主会话。
- **主会话只保留结论和引用**：如果需要溯源，保留子 session 的 ID，而不是把子 session 内容复制过来。
- **给子 Agent 配独立日志**：子 session 日志写到单独文件，调试时按 ID 查，而不是翻主 session。
- **MCP 工具做能力分层**：主 Agent 用读写工具，子 Agent 默认用只读或受限工具。
- **统一返回结构**：`{ ok, summary, refs, error? }` 这样的结构比自由文本更可靠。

## 总结

OpenClaw 的 session 隔离不是简单地把子 Agent 消息从 history 里删掉，而是要让子 Agent 在独立的上下文、工具状态和错误边界里运行。主会话只接收子 Agent 的最终结论，不接收过程。这样才能在复杂自动化任务里保持主 Agent 的决策稳定，也能让调试变得可追溯。

如果你还在被子 Agent 的输出淹没，先检查三件事：session 是否真正隔离、工具是否有共享副作用、错误是否只回传摘要。这三个做到位，大部分污染问题都能解决。

---

