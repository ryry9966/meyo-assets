---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 34805
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw 这类 Agent 编排环境里，助手不再是一次 HTTP 请求就能结束的问答。它可能连接 MCP server、等待人在回路、调用外部 API、运行长任务。一旦链路拉长，就要回答两个问题：这个 agent 还活着吗？任务到底推进到哪一步了？

heartbeat 就是为这个设计的。常见方案有两种：轮询（polling）和推送（push/SSE/WebSocket/webhook）。很多人把二者对立起来，实际工程里更合理的是分层混用。

## 问题

纯轮询的问题：延迟取决于间隔，N 秒一次会让状态感知变钝；多个 agent 同时轮询 MCP 工具时，容易放大服务端压力。

纯推送的问题：SSE/WebSocket 会断，webhook 会丢；连接中断后如果没有兜底，状态可能一直停留在旧值。Agent 如果只相信推送，容易出现“假死”或“跳过关键更新”。

所以取舍不是选边，而是看信号类型和可靠性要求。

## 做法：把心跳分成三层

第一层 liveness：进程/会话是否存活。适合轻量推送，比如每 15 秒一条 SSE ping。  
第二层 progress：任务是否在推进。适合带 sequence 的心跳，更新频率可以更低。  
第三层 dependency：MCP 工具、外部 API 是否健康。适合轮询，因为外部系统不一定支持推送。

以一个 OpenClaw 插件为例，我会这样设计：

- 前端订阅任务状态变化：走 SSE 推送。
- 后台同时维护一个 30 秒间隔的轮询任务，只拉取关键状态摘要。
- MCP server 健康检查每 60 秒轮询一次，失败后指数退避重试。

消息结构尽量简单：

```ts
type Heartbeat = {
  seq: number;          // 单调递增序号
  ts: number;           // 服务端时间
  status: "active" | "stale" | "done";
  progress?: number;
}
```

客户端状态机建议：

`active -> stale -> dead -> recovering`

- 超过 heartbeat interval 未收到且 progress 不变，进入 stale。
- 超过 grace period 仍未恢复，判定 dead，触发重连或重新调度。
- 重连成功后先对账，再恢复消费。

对账（reconcile）是关键：每隔 5-10 分钟，主动轮询一次，用 seq 比对推送链路和轮询链路的结果。不要只依赖推送到达。

## 踩坑点

1. **网关/代理会切断长连接。** 不要依赖 TCP keepalive，SSE 要发注释心跳，WebSocket 要发 ping/pong。
2. **SSE 重连丢事件。** 服务端重启后 Last-Event-ID 可能对不上，导致前端漏掉中间状态。需要保存 last event id 并支持增量续传。
3. **webhook 丢失。** 推送链路天生不保证 exactly-once，网络失败或服务端忙时都会丢。所以一定要有轮询对账，或者把 webhook 做成 at-least-once + 去重。
4. **轮询放大 MCP 压力。** 多个 agent 对同一个 MCP server 高频轮询，会让工具端负载陡增。加 jitter 和本地缓存，状态查询类接口不要穿透到工具端。
5. **长计算被误杀。** 大模型推理、批处理可能长时间没有 progress 更新，不代表任务死掉。心跳里要区分“无进度但存活”，并给长任务设置更大的 grace period。
6. **不要比较墙上时间。** 分布式环境里客户端和服务端时钟可能偏差，判定超时应以服务端 ts 和本地 receive time 结合，最好用 sequence 判断是否连续。

## 可复用建议

- heartbeat 只负责轻量信号，不要塞大 payload；进度详情走单独查询。
- 推送用于“感知变化”，轮询用于“最终一致对账”，二者职责不同。
- 每个 MCP 工具/外部依赖单独配 heartbeat，不要全部复用全局超时。
- 重连退避要有上限，重连后先对账再继续消费。
- 把心跳逻辑做成 OpenClaw 插件或中间件，业务代码不要直接管理连接状态。

## 总结

轮询 vs 推送不是技术选型的二元对立，而是延迟与可靠性之间的分层。推送负责低延迟通知，轮询兜底保证最终一致。真正让系统稳的，是状态机、单调序号和对账机制。做好这些，AI 助手在长任务里才不会因为一个丢失的 webhook 被误判成“假死”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/d389daf6c72405f1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/f109cdf183cb90c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/13d597075b2689b9.png)

