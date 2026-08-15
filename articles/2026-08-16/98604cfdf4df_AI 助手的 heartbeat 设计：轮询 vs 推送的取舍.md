---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 33313
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

在 OpenClaw/Agent/MCP/自动化实践里，heartbeat 不只是“我还活着”。它常常承担三件事：存活探测、状态同步、任务触发。很多项目会自然地从轮询开始：每 10 秒 GET 一次 `/health` 或 `/state`，简单直接。但当远程 MCP server、serverless 插件、多 Agent 协作进来后，轮询的成本和延迟会被放大；于是有人改推送，又发现漏消息、断线、重复触发。

这篇文章不站队，只梳理在 OpenClaw 插件和 MCP 场景下，怎么选、怎么做、哪些坑值得避开。

## 问题

轮询的核心问题不是“土”，而是频率与数据新鲜度绑定。想更快感知状态，就得提高频率；频率一高，远程调用放大、限流、冷启动、日志噪音都出现。推送的核心问题是链路复杂：SSE/WebSocket/webhook 都有断线、重连、乱序、重复投递的问题。如果没有序号和补偿机制，推送很容易从“实时”变成“随机丢”。

## 做法/步骤

1. 先拆 heartbeat 用途。把 liveness、state sync、task dispatch 分开。liveness 用轻量 ping，state 用版本号或快照，task 不要塞进 heartbeat，用事件或队列。
2. 轮询做减法。不要固定 5 秒一次。用 `interval = base * backoff`，加 jitter，比如 10s ± 2s。只拉一个 `status_version`，变化了才拉全量 state。
3. 推送优先 SSE。单向状态流足够时，SSE 比 WebSocket 简单，自动重连、断线续传有 HTTP 基础。需要双向控制再上 WebSocket，但必须自己做应用层心跳和 resync。
4. 每条推送带 `seq` 或 `version`。消费端按 seq 去重，发现缺口主动发起一次 snapshot 拉取。webhook 同理，带 `X-Event-Id` 和签名。
5. 失联判定不要靠一次超时。建议服务端主动下推 heartbeat，客户端 `miss 3 次` 才标记离线；客户端也要有兜底轮询，防止推送链路静默断开。

下面是一个在 OpenClaw 插件里可用的轻量抽象，方便切换通道：

```ts
type Channel = "poll" | "sse" | "ws";

interface HeartbeatChannel {
  start(): void;
  stop(): void;
  onEvent(cb: (ev: HeartbeatEvent) => void): void;
}

// 消费端统一处理
let lastSeq = 0;
onEvent(ev => {
  if (ev.seq <= lastSeq) return; // 去重
  if (ev.seq > lastSeq + 1) {
    fetchSnapshot(); // 补缺口
  }
  lastSeq = ev.seq;
  applyState(ev);
});
```

## 踩坑点

- 把心跳当消息队列：推送通道不可靠，丢事件没有补偿，任务会被静默吞掉。
- WebSocket 只连不校验：连接建立后没有 application-level heartbeat，代理空闲断开不知道。
- 轮询惊群：多个 Agent 或插件同时拉取，加 jitter，避免同一毫秒打爆网关。
- 用客户端本地时间排序：多端时钟不一致，必须用服务端 seq 或服务端时间。
- heartbeat 携带大 payload：本来 200 字节的 ping，带上完整 tools 列表和日志，网络与存储都膨胀。

## 可复用建议

在 OpenClaw/MCP 场景下，我推荐的默认组合是：**低频轮询兜底 + SSE 推送状态 + 任务走独立事件**。不要一上来就 WebSocket；如果已有 webhook 条件，用它但必须实现 idempotency。给所有 heartbeat 事件加 TTL，过期事件直接忽略。把重连和 resync 写成公共模块，而不是每个插件各写一套。

## 总结

轮询不是落后，推送也不是银弹。工程上的取舍在于：你愿意为实时性付出多少重连、幂等和补偿成本。如果状态变化不频繁，轮询加退避完全够用；如果追求秒级感知，就上推送，并把 seq、断线补拉、兜底轮询做扎实。最差的情况是：用推送的方式做轮询，却没有轮询的可靠性。

---

