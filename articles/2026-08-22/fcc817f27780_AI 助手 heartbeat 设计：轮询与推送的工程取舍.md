---
title: AI 助手 heartbeat 设计：轮询与推送的工程取舍
feedId: 34183
source: 综合讨论
publishedAt: 2026-08-22
---

在 OpenClaw 的 agent runtime 里，心跳不是“加分项”，而是长任务可观测性的底线。一个 agent 可能连续跑 20 分钟：中间调用 MCP server、等待子进程输出、驱动插件完成浏览器操作。宿主需要知道它是否还活着，是否卡在某个工具调用上，是否应该发 cancel。

轮询和推送的取舍，本质是两个变量：延迟容忍度、连接维护成本。轮询适合低频状态检查、实现简单；推送适合实时状态流，但要把重连、背压、半开连接处理干净。

## 问题

轮询容易踩的坑不是“轮询不对”，而是所有状态都走同一条重路径。比如 agent 每 1 秒 GET /status，服务端每次查 MySQL，100 个并发 agent 就是 100 QPS 写入同一张心跳表，连接池很快耗尽。推送同样有坑：WebSocket 在 NAT 或代理后可能静默断开，SSE 可能被反向代理缓冲，导致宿主侧看到的是旧数据，等真正发现时任务已经卡了很久。

在 OpenClaw 场景里，还有一个细节：MCP server 的 JSON-RPC 通知是没有响应的。把心跳做成 MCP 请求，服务端一旦繁忙，心跳就会被排在业务请求后面，造成“假死”误判。

## 做法

先契约，后通道。

心跳消息建议保持轻量，字段只放必要的：

```json
{
  "id": "agent_01",
  "seq": 128,
  "status": "running",
  "progress": 0.62,
  "timestamp": 1710000000,
  "ttl": 15
}
```

`seq` 用于乱序检测，`ttl` 是租约时长。宿主侧记录 `last_heartbeat`，超过 `ttl` 才标记 `suspicious`，超过 `2 * ttl` 判死。不要丢一次心跳就杀任务。

然后做通道选型：

| 场景 | 建议 |
| --- | --- |
| 同进程插件/内存 worker | 直接内存 channel，不引入网络心跳 |
| 跨进程/容器，10s 级延迟可接受 | 轮询或 SSE |
| 需要实时状态、常驻连接 | WebSocket |
| 只关心终态 | Webhook/回调 |

如果走轮询，状态缓存优先放 Redis，不要每次查业务库。间隔 5-15 秒足够，超时阈值设置为 `3 * interval`。

如果走推送，客户端必须做应用层 Ping/Pong，并持续维护重连状态：

```text
alive -> reconnect with jitter -> alive
```

## 踩坑

1. 重试风暴：agent 重启后同时重连，gateway 被打满。重连退避要加 jitter，上限建议 30-60s。
2. 心跳与业务共用一个队列：一次大 payload 的工具输出可能把心跳堵在后面。拆成两个通道或给心跳独立优先级。
3. 用客户端墙钟做超时判断：NTP 回拨会误判。优先用服务端时间，客户端用单调时钟做发送间隔。
4. 把 heartbeat 当分布式锁或任务队列：它是租约机制，不保证一致性。

## 可复用建议

- 默认同进程用内存；跨机才考虑网络心跳。
- 保留 `seq` 和 `ttl`，这是最便宜的可观测性元数据。
- 为心跳单独埋点：`heartbeat_latency_p95`、`heartbeat_miss_ratio`、`reconnect_total`。
- 幂等处理：按 `agent_id + seq` 更新状态，重复心跳不产生副作用。
- 对 MCP server 的心跳，用独立轻量通知，不要和业务 JSON-RPC 请求混在一条连接上，必要时单开一条控制连接。

## 总结

轮询不是原罪，推送也不是银弹。工程上先定义清楚超时模型和状态机，再选择通道，最后才优化频率。对 OpenClaw 的 agent 宿主来说，稳定的 heartbeat 能让你少很多“任务明明在跑，宿主却误判超时”的线上事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/bb4efe0a9c704c1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/df2592797620af02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/cc25d8a4283903a0.png)

