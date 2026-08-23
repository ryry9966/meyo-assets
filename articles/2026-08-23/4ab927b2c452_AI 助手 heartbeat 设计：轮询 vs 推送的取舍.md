---
title: AI 助手 heartbeat 设计：轮询 vs 推送的取舍
feedId: 34387
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw、Agent 运行时、MCP 工具或插件自动化里，heartbeat 不只是“我还活着”。它通常承担三类信息：

- 存活（liveness）：进程/组件是否还在运行
- 就绪（readiness）：依赖是否可用，能否接收任务
- 进度/状态变更：任务执行到哪一步、是否失败

很多项目一开始要么无脑加轮询，要么直接上 WebSocket，最后都踩坑。原因是这三个目标对实时性、可靠性和资源开销的要求并不一样。

## 问题

轮询的主要缺陷是：间隔短会导致 CPU 和网络浪费，尤其是 agent 任务本身要跑几分钟时，前端仍可能每秒拉一次全量状态；间隔长又会让状态滞后，关键失败不能被及时发现。

推送的主要缺陷是：需要维护长连接、处理断线重连、事件丢失、背压、鉴权和调试。对 AI 助手场景，很多状态并不是高频变化，但“失败感知”需要尽量快。不同组件对实时性的要求差异很大，不能一套方案通吃。

## 做法/步骤

1. **先区分 heartbeat 类型**  
   不要用一个 `/heartbeat` 同时表达存活、就绪和进度。建议拆成：
   - `GET /healthz`：liveness，只回答“进程在不在”
   - `GET /readyz`：readiness，检查 MCP server、数据库、模型网关等依赖
   - `GET /events?since=last_seq`：状态/进度变更，用于增量同步

2. **默认轮询，按需加推送**  
   在 OpenClaw/Agent 任务场景下，任务进度变化通常以秒到分钟计。前端或编排器用 2–5 秒轮询一个轻量状态接口，已经能覆盖 80% 的需求。  
   只有当 UI 需要实时滚动日志、交互式控制，或者多个组件需要即时协同，才引入 SSE 或 WebSocket。

3. **跨进程/跨主机时分层**  
   - 同一进程内：用宿主事件总线，不要自建轮询线程。
   - 前后端之间：进度流用 SSE，双向控制用 WebSocket。
   - 外部系统回调：用 Webhook，但要加签名、事件 ID 去重和重试。
   - MCP server：stdio 模式下，心跳/日志走 stderr，不要污染 JSON-RPC；HTTP/SSE 模式下，用 `GET /healthz` + `/readyz`，事件用 `/events`。

4. **设计轻量心跳数据契约**  
   心跳包不要塞大 JSON，保留核心字段即可：

```json
{
  "seq": 123,
  "ts": 1710000000,
  "state": "running",
  "progress": 0.42,
  "error": null,
  "version": "1.2.0"
}
```

`seq` 单调递增，`ts` 用 Unix 秒或毫秒，`state` 用枚举。这样客户端可以做增量对比，断线后也能根据 `seq` 补拉 gap。

5. **推送与轮询结合做对账**  
   推送负责“通知”，轮询/增量接口负责“对账”。客户端每次收到推送后更新 `last_seq`；断线重连时不要只重连，还要从 `last_seq` 拉一次增量，避免事件丢失。

## 踩坑点

- **把心跳当成业务状态同步**  
  心跳接口返回一个大任务树，每次轮询都全量序列化，后端压力很大。应该支持 `since` 或 `seq` 做增量。

- **轮询没有退避和抖动**  
  任务失败后所有客户端仍然按原频率打，容易形成重试风暴。轮询间隔可以加随机抖动，长时间无变化时自动降频。

- **推送断线后只重连不补偿**  
  WebSocket/SSE 断线期间的事件会永久丢失。必须保存 `last_seq`，重连后补拉。

- **Webhook 不做幂等和验证**  
  外部回调可能重复、乱序或伪造。需要签名校验，并用 `event_id` 去重。

- **MCP stdio 模式日志污染协议**  
  MCP server 的 stdout 用于 JSON-RPC，任何普通日志输出都会破坏通信。心跳、调试日志必须走 stderr 或文件。

- **长连接被代理/NAT 静默断开**  
  SSE/WebSocket 需要应用层心跳包和合理的空闲超时，否则会出现“连接看着在，实际上已经死了”的情况。

## 可复用建议

- 小项目：只用一个 `GET /api/tasks/:id/status` + 前端 2–5 秒轮询，不要上 WebSocket。
- 中型项目：SSE 推送任务事件，轮询做兜底补偿。客户端维护 `last_seq`，重连后补拉。
- 大型/多实例：用 Redis Streams、NATS 或 Postgres LISTEN/NOTIFY 作为内部事件层，WebSocket 只做边缘分发。
- 所有 heartbeat 端点都设置短超时、简单鉴权、结构化日志。
- 本地调试用 `curl -N` 看 SSE，用 `websocat` 测 WebSocket，能快速定位断连和乱序问题。

## 总结

轮询是基线，推送是优化。先明确你要检测的是“进程是否活着”还是“任务状态是否变化”。能用轮询解决的，不要急着上长连接；只有延迟敏感或交互式控制场景，才引入 SSE/WebSocket/Webhook。混合方案里，让推送负责通知，轮询/增量接口负责对账，是 OpenClaw/Agent/MCP 场景下最稳的工程做法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/589575d48bb59bac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/791f3edaa26ba954.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c4d68e3dfcfd9dfc.png)

