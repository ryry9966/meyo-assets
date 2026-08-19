---
title: AI 助手 heartbeat 设计：轮询与推送的工程取舍
feedId: 33867
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

在 OpenClaw、Agent 与 MCP 插件场景里，heartbeat 经常被当成“万能心跳”：既想探活，又想同步任务状态，还想顺带传点进度。实际落地时，很容易把 heartbeat 做成一个频率不确定、字段不断膨胀、断线后状态混乱的通道。

最近在给一个长时间运行的 Agent 任务加监控时，我们踩了两个典型坑：轮询间隔太短把 MCP server 的 rate limit 打满；推送断线后没有兜底，任务完成事件丢失。整理一下我们的做法和取舍。

## 问题

heartbeat 至少要区分两类需求：

- **liveness**：进程、MCP server、插件宿主是否还活着。
- **progress**：任务是否还在跑、跑到哪一步、是否失败。

如果混在一起，轮询和推送都会变得很难调。

**轮询的问题**：实现简单，但间隔难选。太短会产生明显 QPS。比如 50 个并发任务，3 秒轮询一次，就是约 17 QPS。对于有 rate limit 的 MCP server，这可能直接影响正常 tool call。太长则任务已经完成，用户要等很久才感知到。

**推送的问题**：SSE、WebSocket、webhook 都能降低延迟，但连接管理复杂。SSE 在某些代理/网关下可能被缓冲；WebSocket 要处理鉴权、心跳和断线重连；webhook 则依赖公网可达性。

所以结论不是“选哪个”，而是怎么组合。

## 做法

### 1. 状态机先行

先定义任务状态，不要用 heartbeat 字符串承载业务语义：

```text
QUEUED -> RUNNING -> STREAMING -> SUCCEEDED
                              \-> FAILED
                              \-> CANCELLED
```

heartbeat 只报最小字段：`id`、`status`、`version`、`ts`、`lease_ttl`。其中 `version` 单调递增，用于防止旧事件覆盖新状态。

### 2. 轮询侧：自适应间隔 + 版本校验

不在宿主里写死 `setInterval(3s)`。我们采用：

- `QUEUED` / `RUNNING`：5 秒轮询。
- `STREAMING`：2 秒轮询，因为用户正在等待输出。
- 长时间无状态变化：指数退避到 30 秒。

服务端返回 `version`，客户端可带 `If-None-Match` 或 `version` 参数。重复轮询时，如果 version 没变，就只返回轻量响应，不拉全量上下文。

另外，服务端给每个任务一个 `lease_ttl`。超过 TTL 没有收到客户端心跳，就标记失活。这样可以避免僵尸任务一直占用资源。

### 3. 推送侧：SSE 优先，重连可续传

我们优先用 SSE，原因是 Agent/插件宿主通常已经能处理 HTTP 流式响应。事件格式尽量简单：

```text
event: task.progress
data: {"id":"task_123","status":"RUNNING","version":7,"ts":1718000000}
```

客户端维护 `last_event_id`，断线重连时通过 `Last-Event-ID` 从服务端事件日志续传。这样短断线不会丢事件。

如果 SSE 在网关下不稳定，就退化为 WebSocket。WebSocket 的心跳必须是独立 ping/pong，不要复用业务 heartbeat，否则会引入不必要的状态复杂度。

### 4. 混合策略：推送为主，轮询兜底

线上并不是纯推送或纯轮询。我们采用：

- 正常情况下，推送实时更新。
- 每 30 秒做一次轻量轮询，校验当前 `version` 是否与本地一致。
- 如果推送连接断开超过 60 秒，客户端自动降级为轮询。
- 推送恢复后，停止兜底轮询。

这样既避免推送断线时用户完全失去感知，又不会在推送正常时持续制造轮询压力。

## 踩坑点

1. **把 heartbeat 当进度上报**  
   每 1 秒推一次完整上下文，连接很快被大 JSON 撑满。进度字段应该是可选的小对象，不要塞日志、堆栈或完整 prompt。

2. **客户端时间不可信**  
   任务超时判断必须用服务端生成的 `ts`。如果客户端本地时钟偏移，很容易把正常运行的任务误判为超时。

3. **推送重连风暴**  
   断线后如果没有退避和 jitter，几十个客户端会在同一时刻重连。重连间隔要指数退避 + 随机抖动，例如 `1s, 2s, 4s, 8s, max 30s`。

4. **状态回退**  
   重复推送或延迟事件可能携带旧 version。客户端必须忽略 `version <= currentVersion` 的事件，否则任务会从 `SUCCEEDED` 回退到 `RUNNING`。

5. **OpenClaw 插件宿主里滥用定时器**  
   直接用多个 `setInterval` 访问 MCP 会放大 token 占用和并发连接。建议用一个统一调度器，按任务阶段动态调整频率，并对轮询产生的 token/QPS 做统计。

## 可复用建议

- **字段最小化**：`id, status, version, ts, lease_ttl`，进度另放可选字段。
- **liveness 与 progress 分离**：liveness 低频 ping，progress 才进入高频通道。
- **推送优先，轮询兜底**：不要二选一。
- **version 单调递增**：客户端永远忽略旧版本。
- **服务端负责时间戳**：不做客户端对时。
- **加上 metrics**：`heartbeat_latency`、`poll_qps`、`reconnect_count`、`stale_event_dropped`，这些指标比日志更容易发现 heartbeat 设计问题。

## 总结

heartbeat 不是越频繁越好，也不应该成为业务数据的传输通道。工程上更稳妥的做法是：

**推送优先降低延迟，轮询兜底保证可恢复，状态机约束语义，version 防止回退，lease TTL 清理僵尸任务。**

这样实现看起来比“每 3 秒轮询一次”复杂一点，但它能在 MCP server 限流、网络抖动、插件宿主重启时保持行为可预期，而不会让 heartbeat 本身变成新的故障源。

---

