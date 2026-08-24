---
title: AI 助手 heartbeat 设计：轮询、推送与兜底对账
feedId: 34591
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 的 Agent 经常要连接外部系统：任务队列、MCP server、插件状态、自动化流水线。Agent 需要知道“任务完成了吗”“资源变更了吗”“连接还活着吗”。heartbeat 就是这类周期性或事件驱动的状态感知机制。工程上通常分成两种：轮询（polling）和推送（push / webhook / SSE / WebSocket）。

## 问题

轮询实现简单，但不实时，且会产生大量无效请求；推送实时性高，但引入长连接、重连、去重和背压问题。很多项目一开始无脑轮询，等秒级需求上来后又硬改推送，结果两边都留下坑。关键不是选谁，而是按频率、实时性和可靠性要求设计组合。

## 做法

**第一步：先给事件分级。**  
例如存活心跳 30s、进度更新 10s、完成事件最好 1s 内。不同级别可以走不同通道，不要用同一个机制硬扛所有场景。

**第二步：低频/非实时用轮询。**  
以 MCP 资源读取为例，定时调用 `readResource` 检查版本号或状态字段。不要每次拉全量，优先使用 ETag/If-None-Match；API 不支持的，返回轻量 status 字段。轮询必须加 jitter，避免多实例同时打同一个源。

```javascript
async function pollStatus() {
  const res = await fetch(url, {
    headers: { 'If-None-Match': etag },
  });
  if (res.status === 304) return;
  updateState(await res.json());
}
setInterval(pollStatus, 5000 + Math.random() * 1000);
```

**第三步：高频/实时用推送。**  
MCP 的 streamable HTTP 或 SSE endpoint 可以主动推事件；WebSocket 适合双向交互。推送必须带事件序号 `seq`，消费者只接受 `seq > lastSeq` 的事件，处理完持久化 checkpoint。

```javascript
ws.on('message', (event) => {
  if (event.seq <= lastSeq) return; // 重复事件
  handle(event);
  lastSeq = event.seq;
  saveCheckpoint(lastSeq);
});
```

**第四步：混合兜底。**  
即使有推送，也保留低频轮询对账。例如每 60s 拉一次任务列表和本地状态对比，发现缺失就补拉。这样推送断线不会导致永久丢事件。

## 踩坑点

- **轮询间隔太短**：容易触发 API 限流或数据库压力，尤其多实例部署时所有实例同时轮询。加 jitter 和共享缓存。
- **推送连接被静默断开**：经过 Nginx/负载均衡时，默认 `proxy_read_timeout` 可能 60s 就断空闲连接。必须做应用层心跳，别依赖 TCP keepalive。
- **重连恢复位置错误**：SSE 的 `Last-Event-ID` 和 WebSocket 自定义 seq 要设计清楚，否则重连后从错误位置恢复，造成重复或丢失。
- **推送消费积压**：消费者处理慢导致队列堆积、内存上涨。要设置背压：超过 N 条未确认就暂停或丢弃旧事件，并记录 lag。
- **用本地时间比较新旧**：容器时钟漂移会坑人，统一使用服务端 seq 或事件时间戳。
- **MCP stdio transport**：通常是一问一答，不适合服务端主动推送；需要推送时优先评估 streamable HTTP/SSE。

## 可复用建议

- 默认轮询，只有明确需要秒级实时才引入推送。
- 轮询统一封装：条件请求、指数退避、jitter、超时。
- 推送必须可重放：seq、checkpoint、幂等消费。
- 做一个 heartbeat 健康面：暴露 metrics，如空轮询比例、推送重连次数、事件 lag、p99 同步延迟。
- 把间隔、超时、重试次数做成配置，不同环境可调。

## 总结

heartbeat 不是轮询和推送的二选一，而是“推送优先、轮询兜底、可重放可对账”的组合。先按事件频率和实时性分级，再决定通道；无论哪种方案，都要把重复、丢失和积压当成默认会发生的问题来设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1ed2c297b9134d7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/51f0bbe225d431b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8545811f20434521.png)

