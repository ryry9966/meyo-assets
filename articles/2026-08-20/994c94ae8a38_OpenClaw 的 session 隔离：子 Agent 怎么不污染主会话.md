---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 33891
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 里用子 Agent 处理复杂任务已经很常见：让一个子 Agent 去做代码审查、让另一个去跑多步搜索、再让一个去调用 MCP 工具完成数据清洗。子 Agent 的价值在于把长链路拆出去，避免主 Agent 在一个任务里反复绕圈。

但实际跑起来后，很多人会遇到一个隐蔽问题：主会话越来越“脏”。

表现为：主 Agent 开始引用子任务里的中间报错；上下文窗口很快被内部日志占满；重试时旧的工具返回残留在对话里；用户跟主 Agent 对话时，它突然冒出一句“刚才第 3 步的 JSON 解析失败”。这些都不是玄学，而是子 Agent 的执行过程被默认写进了主会话历史。

## 问题本质

OpenClaw 的 Agent 运行时会维护 session 上下文。创建子 Agent 时，如果不显式指定隔离策略，子 Agent 的推理步骤、工具调用、错误信息、MCP 返回片段，会作为普通消息追加到父 session 或共享事件流里。

后果有三类：

1. **上下文膨胀**：中间过程占用大量 token，主 Agent 的有效记忆被稀释。
2. **行为漂移**：子任务的局部失败被主 Agent 当成当前任务状态，导致错误重试或错误回答。
3. **审计困难**：无法区分哪些信息来自用户、哪些来自子 Agent 内部。

所以 session 隔离不是“把子 Agent 放后台跑”那么简单，而是要控制**执行边界、事件回传、生命周期**三个层面。

## 做法 / 步骤

### 1. 给子 Agent 独立 session context

创建子 Agent 时不要让它继承父 session 的完整历史。显式传入独立的 `sessionId` 或 `contextKey`，只把必要的任务描述传进去。

```ts
const sub = await spawnAgent("reviewer", {
  session: {
    isolate: true,
    parentId: currentSession.id,
    maxTurns: 10,
    ttl: 180 // 秒
  }
});
```

这里 `isolate: true` 表示子 Agent 的中间过程不会直接写入父 session。`parentId` 保留溯源关系，方便后续审计。

### 2. 控制回传内容

子 Agent 执行完，不要让它把完整历史返回给主 Agent。只回传结构化结果：

```ts
const result = await sub.run(task);
await sessionBridge.ingest({
  source: "subagent:reviewer",
  status: result.status,
  summary: result.summary,        // 3-5 条关键结论
  artifacts: result.artifactPaths // 文件路径或外部引用
});
```

主 session 里只多一条摘要消息，而不是几十条内部日志。

### 3. 过滤事件流

OpenClaw 的事件总线可能会把子 Agent 的工具调用事件广播到父 session。需要在父 Agent 的事件处理里加来源过滤：

```ts
bus.on("agent:message", (msg) => {
  if (msg.source.startsWith("subagent:") && msg.type !== "final") {
    return; // 丢弃子 Agent 的中间消息
  }
  parentSession.append(msg);
});
```

这个过滤必须在事件进入父 session 之前完成，否则等于没做。

### 4. 设置 TTL 与清理

子 session 完成后要主动关闭，避免残留。用 `try/finally` 确保异常时也能清理：

```ts
let subSessionId: string | null = null;
try {
  const sub = await spawnAgent(...);
  subSessionId = sub.session.id;
  const result = await sub.run(task);
  return result.summary;
} finally {
  if (subSessionId) {
    await sessionManager.close(subSessionId, { reason: "completed" });
  }
}
```

## 踩坑点

1. **事件冒泡没过滤干净**：MCP 工具调用产生的日志可能通过全局 logger 直接写入主会话。需要给子 Agent 传一个 contextual logger，把日志重定向到子 session 自己的缓冲区，而不是全局输出。

2. **流式输出被直接转发**：如果子 Agent 启用了流式返回，而父 Agent 把每个 chunk 都当成消息处理，主会话照样会膨胀。正确做法是缓存子 Agent 的流式输出，只在结束时回传 final message。

3. **超时后父 session 残留 pending 状态**：子 Agent 崩溃或超时后，如果父 session 里还挂着一个“等待子任务返回”的状态，后续对话会一直带着这个半死状态。必须在 `finally` 里显式清理，并给父 session 写入一个明确的“子任务失败/超时”摘要。

4. **过度隔离导致主 Agent 无法复核**：完全隔离后，主 Agent 只拿到一句“任务完成”，但无法判断子 Agent 是否做对了。建议保留结构化摘要 + 关键文件路径，必要时让主 Agent 可以按路径读取产物，而不是把原始日志灌回来。

## 可复用建议

- 封装一个 `runIsolatedSubagent` 工具函数，把 session 创建、事件过滤、结果回传、清理逻辑统一起来。以后所有子 Agent 调用都走这个入口，避免每个插件各写一套。
- 给消息打来源标签：`source: "subagent:reviewer"`、`source: "user"`、`source: "mcp:github"`。这样后续做上下文裁剪时，可以优先淘汰子 Agent 的中间消息。
- 主 session 设置软上限，到达阈值后优先裁剪 `source=subagent:*` 且 `type != final` 的消息。
- 子 Agent 的返回体设计成 JSON 摘要 + artifact 引用，不要返回原始日志全文。

## 总结

OpenClaw 的 session 隔离不是简单的开关位，而是一套组合动作：独立 context、事件过滤、结构化回传、生命周期清理。做好之后，主会话会干净很多，子 Agent 的中间失败不会再干扰主 Agent 的判断；同时保留必要的溯源和产物引用，主 Agent 仍然可以复核子任务结果。

如果你的 OpenClaw 主会话最近越来越“健忘”或“情绪不稳定”，先别急着调温度，检查一下子 Agent 是不是把内部过程都吐进了主 session。

---

