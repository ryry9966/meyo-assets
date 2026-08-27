---
title: OpenClaw 子 Agent 会话隔离：别让自动化把主线程搞脏
feedId: 34988
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 上跑自动化，子 Agent 是绕不开的能力。无论是让一个子 Agent 去查资料、跑 MCP 工具，还是执行一段插件逻辑，主 Agent 都能把任务拆出去，自己继续做编排。但很多实践者会发现一个尴尬现象：自动化跑了几轮后，主会话变得非常“脏”——上下文里塞满了子 Agent 的中间推理、工具输出、重试日志，甚至错误堆栈。主 Agent 开始引用不该引用的话，token 消耗成倍上涨，多子 Agent 并发时还会串线。

## 问题

核心矛盾是：子 Agent 需要上下文才能干活，但它的“过程上下文”不等于主会话需要保留的“结果上下文”。默认情况下，子 Agent 往往继承当前 conversation history，或者执行结束后把完整消息流 merge 回主线程。这会导致：

- 主 Agent 的决策被子 Agent 的中间态干扰；
- 每个子任务的过程都进主历史，token 随任务数线性甚至指数增长；
- 子 Agent 的工具副作用（写文件、发消息、改配置）直接落在共享环境；
- 错误堆栈和调试信息进入主会话，既污染又可能泄露敏感路径。

## 做法 / 步骤

我在 OpenClaw 上采用的隔离方式，核心是“子 Agent 作为黑盒执行单元，主 Agent 只消费结构化结果”。

**1. 创建子 Agent 时强制独立 session**

不要默认 spawn 到当前 thread。给子 Agent 开一个新的 session/thread，只传任务描述和必要输入，不携带主历史。伪代码大概是这样：

```ts
const sub = await openclaw.spawn({
  agent: 'researcher',
  prompt: task,
  session: {
    mode: 'isolated',
    ttl: 300,
    parent: mainSession.id
  }
});
const result = await sub.done();
return {
  summary: result.summary,
  artifacts: result.artifacts,
  confidence: result.confidence
};
```

关键是 `mode: 'isolated'` 和 `parent` 分开：父子关系只用于追踪，不用于上下文继承。

**2. 子 Agent 只回传结构化结果**

子 Agent 的最终输出不是自然语言长文，而是 JSON 或固定结构。例如：

```json
{
  "status": "ok",
  "summary": "取到 3 条数据，其中 2 条有效",
  "artifacts": ["run_20250601_001.json"],
  "confidence": 0.85
}
```

主 Agent 拿到这个对象后，自己决定要不要展开引用。过程消息、工具调用日志、中间推理都留在子 session 里，不 merge 回主历史。

**3. 工具和文件系统按 session 隔离**

如果子 Agent 会使用 MCP 工具或读写文件，必须保证它不直接操作主工作区。可以用 session id 作为命名空间：

```
workspace/runs/{session_id}/input/
workspace/runs/{session_id}/output/
```

MCP server 如果支持 per-session 实例，优先使用；如果只支持共享实例，确保工具是只读或对副作用做显式白名单。不要在子 Agent 里允许“发送消息”“修改全局配置”这类无边界操作。

**4. 设置 TTL 和清理**

子 session 要设置存活时间，结束后主动清理临时文件、缓存和消息历史。否则大量僵尸 session 会拖慢存储和检索。

## 踩坑点

- **只隔离对话，不隔离副作用**：最常见。子 Agent 虽然开了新 session，但工具调用写的是共享目录，或者 MCP server 是单例，结果还是把主环境搞脏。
- **错误堆栈直接回传**：子 Agent 异常时，如果直接把 trace 返回给主 Agent，等于把内部路径、依赖版本甚至部分配置打进主历史。需要截断或只返回错误码和简短原因。
- **嵌套子 Agent 失控**：子 Agent 又 spawn 子 Agent，session 树越来越深，最后排障极其困难。建议限制最大深度，超过就合并结果或直接失败。
- **并发子 Agent 共用工作区**：多个子 Agent 同时写一个目录或同一个 MCP 资源，会互相覆盖。每个子 Agent 必须有独立命名空间。

## 可复用建议

- 子 Agent 接口设计成“纯函数式”：输入明确、输出结构化、副作用通过 artifacts 显式返回。
- 主 session 只保留“决策、引用、结论”，不保留子 Agent 过程。
- 使用 run_id 贯穿任务：主 session 生成 run_id，子 Agent 所有产物和日志以 run_id 命名，方便追踪但天然隔离。
- 监控每个主 session 的子任务 token 占比，如果子过程 token 超过主决策 token，说明隔离不够。
- 子 Agent 的错误处理要单独设计，区分“用户可读错误”和“内部调试信息”，后者只写日志，不进主会话。

## 总结

OpenClaw 的 session 隔离不是简单的“开个新窗口”。真正有效的隔离需要同时切断上下文继承、工具副作用、文件系统共享和错误信息回流。把子 Agent 当作一个短暂存在的黑盒，主 Agent 只接收结构化结果，才能让自动化稳定、可维护，也不会把主会话越跑越脏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/4c574ac6220c6afd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ce191cbc3e72cef4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8dd95347b0ef571b.png)

