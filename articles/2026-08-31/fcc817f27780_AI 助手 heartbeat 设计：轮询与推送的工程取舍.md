---
title: AI 助手 heartbeat 设计：轮询与推送的工程取舍
feedId: 35459
source: 综合讨论
publishedAt: 2026-08-31
---

在 OpenClaw 的 Agent / MCP / 插件开发里，heartbeat 几乎绕不开：你需要等一个外部任务完成，比如 CI、审批流、长耗时 MCP 工具调用、自动化脚本跑批。宿主进程不能一直阻塞，于是只能让 AI 助手主动或被动地感知状态变化。

但很多实现一开始就走偏：有人把轮询间隔设成 1 秒，远端限流、本地 CPU 空转；有人上了 WebSocket，结果连接假死没人发现，任务完成事件丢了半个小时后才靠人工发现。

这篇不讨论“哪个更好”，只写工程上怎么选、怎么组合。

## 背景与问题

Heartbeat 的职责通常有三类：

1. **保活**：证明通道或插件进程还活着。
2. **状态同步**：周期性拉取远端任务进度。
3. **完成通知**：任务结束或关键状态变化时立即触发。

轮询的优势是实现简单、不依赖公网可达、天然适合本地或内网环境。缺点是空转请求多、延迟受间隔限制、远端压力大。推送的优点是延迟低、无效请求少，但需要维护连接、处理重连、幂等、可达性等问题。

真正的问题不是“轮询还是推送”，而是：哪些状态变化值得推送，哪些用低频轮询兜底就够了。

## 做法：先稳轮询，再叠推送

### 1. 别用 `setInterval` 直接包异步任务

这是最常见的坑。`setInterval` 不会等上一次异步执行结束，如果 `fetchStatus` 或 `handle` 执行时间超过间隔，回调会堆积，甚至出现多个请求同时打到远端。

更稳的做法是用 `setTimeout` 链式调度，并给每次请求加超时：

```ts
async function poll() {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const state = await fetchStatus(signal);
    await handle(state);
  } catch (err) {
    if (isAbort(err)) markTimeout();
    else markError(err);
  } finally {
    clearTimeout(timer);
    timer = setTimeout(poll, intervalMs);
  }
}
```

轮询间隔建议放 5–15 秒作为兜底，不要默认 1 秒。对大多数自动化任务，这个延迟是可接受的。

### 2. 明确推送通道的适用边界

如果 OpenClaw 插件或 MCP server 支持 SSE / WebSocket / webhook，优先用推送来触发“即时检查”，而不是提高轮询频率。

- **本地插件**：可以直接开一个 WebSocket 或 IPC 通道，宿主到插件的通信延迟很低。
- **外部服务回调**：用 webhook 常见，但本地开发时往往不可达，需要隧道，同时注意验签。
- **MCP transport**：目前常见的是 streamable HTTP 或 SSE，SSE 适合单向事件流，WebSocket 适合双向控制。

### 3. 混合策略：推送触发 + 低频轮询兜底

推荐结构：

- 低频轮询（例如 8 秒）作为 baseline，保证任何推送通道故障时不会完全失明。
- 推送消息只携带 `eventId` 或 `sequence`，客户端收到后立即拉取全量状态或直接处理。
- 心跳包与业务事件分离：心跳只证明通道存活，不携带大 payload。

这样即使推送延迟、断线、丢消息，低频轮询也能最终一致。

## 踩坑点

1. **轮询间隔太激进**  
   远端 API 限流、本地 CPU 空转、日志爆炸。5–15 秒通常足够。

2. **异步任务堆积**  
   `setInterval` 包异步任务，或者 `setTimeout` 递归里没 `await` 就调度下一次，都会导致并发雪崩。

3. **推送通道假死**  
   WebSocket 连接半开时，应用层没有心跳，客户端和服务器都以为连接正常，实际数据已经不流通。需要 ping/pong 或定时心跳，并设置超时重连。

4. **断线重连丢事件**  
   重连后如果只拿到最新状态，中间状态可能丢失，导致 Agent 误判。建议所有事件带单调递增 `sequence`，客户端记录最后消费位置，重连后从断点续传或显式全量对账。

5. **webhook 内网不可达**  
   本地开发时 webhook 收不到，需要隧道工具，同时增加签名校验防止伪造。

6. **推送流量过大导致背压**  
   事件推得太快，客户端处理不过来，队列无限增长。需要在客户端做背压控制，或让服务端合并事件。

## 可复用建议

- **默认低频轮询 5–10 秒**，关键状态变化才上推送。
- **心跳与业务分离**：心跳只表示通道存活，不承载状态。
- **事件带 `sequence` 或 `eventId`**，客户端持久化 cursor，重连后断点续传或全量对账。
- **重连使用指数退避 + 随机抖动**，避免大量客户端同时重连造成惊群。
- **监控三个指标**：轮询空转率、推送延迟、重连次数。空转率高说明事件密度低，可以降低轮询频率或改成纯推送；重连次数高说明连接治理需要调整。
- **把 heartbeat 封装成独立 service**，不要和业务逻辑混在一起，方便测试、替换和降级。

## 总结

AI 助手的 heartbeat 设计，核心不是轮询与推送二选一，而是分层：低频轮询兜底，推送负责关键事件，sequence 保证一致性，退避重连保证韧性。先让系统简单可观测，再按真实延迟和资源指标决定要不要上推送。

对 OpenClaw 场景来说，很多任务根本不需要毫秒级实时，8 秒一次的轮询就够；只有审批、人工介入、长耗时任务完成这类关键变化，才值得引入推送。控制复杂度，比追“实时”更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/fe8a44bd4fbbca1e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/faeec65a9bff30e9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/502402d120cf077c.png)

