---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 35131
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw / Agent / MCP / 插件这类自动化环境里，heartbeat 通常要回答三个问题：

1. 任务还活着吗？
2. 当前执行到哪一步？
3. 下游工具调用是不是卡住了？

很多开发者一开始就在“轮询还是推送”上纠结，但实际工程里，这取决于 heartbeat 的语义和链路长度。如果选错，要么前端每 1 秒轮询 `/status`，后端为了返回最新状态不断扫表、抢锁，CPU 和日志噪音一起上涨；要么强行上 WebSocket 推送，结果又要处理鉴权、重连、反压、多实例广播，复杂度远超收益。

问题不是“哪个更好”，而是“在什么场景下用哪种”。

## 问题

最常见的错误是把“存活检测”和“进度推送”耦合在同一个 heartbeat 里。

例如 agent 每 500ms 推一次 `still running`，前端既要判断是否活着，又要从中提取真实进度，结果真实事件被心跳淹没。另一种情况是，开发者用 WebSocket 做单向状态流，但客户端重连、代理超时、断线续传都要自己写，最后发现不如 HTTP 简单可靠。

轮询的问题是延迟高、无效请求多；推送的问题是连接状态复杂、排障难。OpenClaw 生态里大量组件是本地进程或插件，其实并不需要把心跳设计得像分布式消息系统那么重。

## 做法

建议按语义分层，不要一个 heartbeat 包打天下。

**1. 拆分 heartbeat 类型**

- **存活心跳（liveness）**：只回答“进程/任务是否还活着”。
- **进度事件（progress event）**：描述“现在执行到哪一步”，可以带工具调用、step 变更。
- **租约续期（lease keepalive）**：向调度器证明“我还持有这个任务”，过期则允许重试或接管。

**2. 存活检测用轻量轮询**

进程内 agent 可以用文件 mtime、pid 或 Unix socket；跨进程用极轻量 UDP ping 或 `/healthz`。间隔 3–5 秒足够，不要每 200ms 打一次。

**3. 进度状态优先用“带游标的增量轮询”**

不要每次全量拉状态。服务端返回单调递增的 `seq` 和变更事件，客户端请求 `?since=lastSeq`，只拿增量。这样轮询的成本接近推送，还天然支持断线重放。

```http
GET /tasks/42/events?cursor=1042

id: 1043
data: {"type":"tool_start","tool":"browser","ts":1716900000}
```

**4. 需要低延迟时选 SSE，而不是 WebSocket**

SSE 单向、基于 HTTP、自动重连、可带 `Last-Event-ID`，对状态流来说足够。在 OpenClaw 插件网关里暴露 `/tasks/{id}/events`，每 10–15 秒发一条注释心跳 `: hb`，防止云网关或 nginx 把空闲连接断开。

**5. 设置显式租约过期**

agent 每次写心跳时更新 `lease_until`。调度器只判断租约是否过期，不依赖连接状态。连接断了但租约还没过期，任务不应被立即判死。

## 踩坑点

- **把 heartbeat 当进度更新**：心跳只携带 `ts` 和 `seq`，不要放业务字段。否则客户端需要写过滤逻辑，状态恢复也很脏。
- **代理静默断开**：长连接没有应用层心跳时，云负载均衡、nginx、家庭路由器都可能在 15–30 秒空闲后切断。SSE 注释心跳比空数据帧更省流量。
- **重连风暴**：推送断开后，所有客户端同时重连，容易打挂网关。需要 jitter 和指数退避，例如 `Math.random() * 1000 + 2 ** attempt * 1000`。
- **多实例丢事件**：事件只放在进程内存 pub/sub 里，重启或重连后 cursor 对不上。可以使用持久化事件日志，或 Redis Stream 保留最近 N 条事件。
- **心跳本身变成热点**：大量 agent 同时写同一张表或同一个文件，会造成锁竞争。可以分片、批量写，或用只追加的 pings 替代 upsert。

## 可复用建议

- 默认方案：单机插件用文件 mtime + 2 秒增量轮询；跨网络用 SSE + 游标。
- 不要一上来就引入 Kafka / Redis Stream，除非你已经在用。本地 agent 的规模通常不需要那么重的基础设施。
- 把 heartbeat 当成观测信号：记录 `heartbeat_latency_seconds`、`missed_heartbeats_total`、`reconnect_delay_seconds`，比翻日志更早发现问题。
- 如果下游工具调用可能卡住，除了 heartbeat，还要给每个 step 设置 timeout 和 cancellation token。否则心跳正常，任务照样卡死。

## 总结

轮询与推送不是二选一，而是按语义分层：

- 存活用轻量轮询；
- 进度用增量轮询；
- 实时事件用 SSE；
- 任务持有权用显式租约过期。

这套组合在 OpenClaw 这类本地 Agent / 插件环境里最省心，排障路径也清晰。心跳设计得越简单，越不容易变成新的故障源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/5d2e28ae825e8b94.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/85ed89e2fa81c4f2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/72dc122cfebd68b9.png)

