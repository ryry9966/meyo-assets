---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的工程取舍
feedId: 31906
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当 AI 助手跑起长任务时，你在等什么？

在 OpenClaw 的 Agent 实践里，我们经常让模型通过 MCP 插件执行耗时操作：查数据库、调外部 API、运行脚本。这些任务短则数秒，长则数分钟。而用户可能关掉 Web 页面、切到移动端后台，或者单纯等得不耐烦点了“停止生成”——这时跑着的 Agent 就进退两难：继续算浪费资源，立即停又需要及时感知。

**这就是心跳（heartbeat）要解决的问题**：让 AI 服务端和客户端互相确认“我还在”，如果一方掉线，另一方应及时止损并释放上下文。

在实现上，最大的分歧就是“**谁来问**”：
- 轮询（polling）：客户端定期发 HTTP 请求，服务端回答“任务还在跑”或返回结果。
- 推送（push）：通过 WebSocket、SSE 等长连接，服务端主动下发心跳或状态变更。

我们手头有 MCP 的 transport、Agent 的 long‑running tool 机制、以及各种部署环境（云函数、容器、边缘节点），选错策略会直接带来连接断开、资源爆炸、客户端假死等工程痛感。下面聊清楚取舍。

---

## 问题拆解：不是选一个，是拼一个状态机

纯轮询最简单：`setInterval(ping, 3000)`，服务端返回 `{"alive": true}`。但缺陷明显：
- **无效流量**：任务跑很久却没变化时，大量空转。
- **延迟感知**：用户中断操作的信号要等到下一次轮询才能送达，最差延迟等于轮询间隔。
- **后端压力**：高并发场景下，每 3 秒一次 ping 能让 API Gateway 的计量器疯涨。

纯推送看似优雅，实际对网络中间件极不友好：
- 反向代理（nginx、ALB）默认长连接超时 60-120 秒，无业务数据经过时会被沉默切断，客户端却不知道。
- 移动端、弱网环境经常断连，推送通道重建成本高。
- Serverless 平台（如 Cloudflare Workers、部分函数计算）压根不支持 WebSocket 长连接升舱，只能使用 WebSocket Hibernation 或退化到 HTTP。

因此，工程上我们往往是**混合策略 + 状态机**，而非二选一。

---

## 做法与步骤：OpenClaw 插件里的可插拔心跳设计

### 1. 传输层协商
在 MCP server/client 建立连接时，能力协商可以加入心跳模式：
```
capabilities:
  heartbeat:
    modes: ["websocket_ping", "http_polling"]
    preferred_interval_ms: 15000
```
- 支持 WebSocket 的环境优先用 `websocket_ping`（帧级别 ping/pong，几乎零开销）。
- 不支持自动降级到 `http_polling`，通知客户端进入轮询模式。

### 2. 心跳包约定
不管用哪种传输，心跳包应极简且幂等。例如 MCP JSON‑RPC 的 `ping` method 可直接复用：
```json
{"jsonrpc":"2.0","method":"ping","id":"hb-1713","params":{"nonce":"a1b2c3"}}
```
服务端需回 Pong，携带服务侧最新任务状态（`running` / `cancelled`）。如果任务已完成，直接在 Pong 中夹带结果引用，减少额外的拉取请求。

### 3. 状态机设计（Agent 侧）
Agent 长任务采用 context‑based 模型：
- 启动任务时，创建一个 `heartbeatCtx`，携带 cancel 函数。
- 每次收到心跳请求，检查 `heartbeatCtx` 是否仍活跃；若超时未收到，自动 cancel，触发 `tool` 内部的 abort 逻辑。
- 客户端显式“停止”动作也通过心跳通道传入取消信号，响应在 1 秒内终止。

### 4. 自适应与熔断
轮询模式下加入退避：
- 初始间隔 3 秒，连续 5 次无变化后拉长到 10 秒。
- 服务端返回 `"throttle"` 信号或 429 时，客户端强制拉长间隔到最低 30 秒。
- 同时在监控中记录：`heartbeat_interval_actual`、`push_drop_rate`。

---

## 踩坑点实录

1. **代理超时比你想象得快**
   ALB/nlb 默认 idle timeout 60s，防火墙可能更短。如果 WebSocket 只在有业务变化时才推消息，空闲 55 秒后就断连。**解决**：服务端主动发空 ping 帧，间隔必须小于链路最小 idle timeout（通常设为 30 秒）。

2. **移动端休眠导致“僵尸连接”**
   App 退后台后，WebSocket 可能被系统挂起，恢复后旧连接已坏死但客户端无感知。**解决**：结合页面 Visibility API，回到前台时立即发一次心跳并重置超时计时，服务端对长时间无心跳的连接主动 close。

3. **Serverless 的 WebSocket 陷阱**
   某些平台宣称支持 WebSocket，实则只允许在请求处理期间保持连接，函数 return 后连接断开。**务必验证**：在函数入口用 `context.callbackWaitsForEmptyEventLoop = false`，并通过平台专用 API 维持连接（如 API Gateway WebSocket API + 持久化 DynamoDB connectionId）。

4. **轮询风暴与缓存**
   轮询时客户端重复请求同一个任务状态，如果没加缓存或去重，后端可能被打成线性增长。**解决**：服务端对短时间内的相同查询使用内存缓存或 Etag；客户端带上 `If-None-Match`，重复时返回 304。

---

## 可复用建议

- **统一心跳接口**：在 OpenClaw 插件层抽象一个 `HeartbeatProvider`，它可以使用 MCP 的 `ping` 或自定义 HTTP endpoint，包含连接保活、取消信号、结果携带三大职责。
- **监控四板斧**：`hb_latency_p99`、`hb_miss_rate`（连续 3 次未收到视为断连）、`reconnect_count`、`polling_degraded`（降级比例）。这些指标比“CPU 高”更能直接反映体验。
- **给 Agent 留一个“逃生通道”**：在长 tool 的内部循环中加入 `select` 检查心跳 context 的 `Done()`，保证取消信号能在毫秒级生效，而不是等整个 batch 跑完。
- **文档写清楚 Fallback 行为**：让你的插件使用者知道，在浏览器支持 WebSocket 时会走推送，在 curl 或脚本调试时会走轮询。并提供环境变量显式切换，例如 `HEARTBEAT_MODE=force_polling`，方便压测。

---

## 总结

Heartbeat 不是什么新话题，但在 Agent 和 MCP 插件环境下，它被赋予了“**感知用户留存+传递取消信号**”的双重重任。轮询保底、推送提效是务实路线；关键是把状态机、超时重试、降级熔断、可观测性一起设计，而不是等到用户投诉“任务卡死了”再去打补丁。

下一版你可以试试把心跳包从“是否活着”升级成“活着且进度 n%”，成本增加不多，但端侧的等待体验会好很多。

---

