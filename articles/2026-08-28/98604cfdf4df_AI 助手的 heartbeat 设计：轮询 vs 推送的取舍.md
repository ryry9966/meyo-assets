---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 34954
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw、Agent、MCP 插件和自动化调度链里，一个很常见的故障形态不是“进程退出”，而是“连接还在，但已经不干活了”。例如：

- MCP server 进程还挂着，stdio 管道也没断，但内部事件循环被同步日志写满；
- WebSocket 连接没断，但远端插件宿主卡在某个阻塞调用里；
- 浏览器扩展/网关能建连，但消息队列堆积，响应延迟从 2s 变成 30s。

这时候单纯监控 TCP 连接、stdin/stdout 是否存在，是发现不了问题的。需要引入 heartbeat，用“探活”来区分“慢”和“死”。

heartbeat 本质上不是证明“进程在”，而是证明“事件循环还能响应请求”。这决定了它不能只做传输层探测，最好要落在应用协议或 MCP 的 JSON-RPC 层。

## 问题：轮询与推送的边界

轮询方案实现简单，但延迟和开销不可兼得。间隔短，探活及时但请求多；间隔长，省资源但可能等很久才发现假死。

推送方案及时，但要求服务端支持主动发帧，并且客户端要维护读超时。在 Nginx、L7 网关、某些浏览器代理后面，SSE/WebSocket 容易被缓冲或静默断开。所以推送也不是“开了就稳”。

工程上需要根据部署形态选型：

- stdio / 子进程类 MCP：优先轮询，因为子进程很难推送到父进程。
- HTTP / WebSocket / SSE 类插件或网关：优先推送，或“推送为主 + 本地轮询兜底”。
- 短期任务 / Agent 单次调用：可以不做常驻 heartbeat，只做端到端超时。

## 做法 / 步骤

### 1. 先定义 heartbeat 语义

不要把 heartbeat 做成“空包就算活”。对 MCP 来说，协议内通常有 `ping` 或 no-op 请求，应优先使用。对自定义插件，可以约定一个轻量 `health` 事件，但不要让它执行业务逻辑。

建议区分两层：

- **Liveness：传输层是否可通信**。例如 WebSocket 是否还能收发，stdio 是否可写。
- **Readiness：应用层是否能处理业务**。例如队列深度、上一次成功执行时间、是否能在 1s 内返回 no-op。

只有 readiness 失败才应该触发重启或摘流。liveness 失败只是重连。

### 2. 轮询方案

客户端定期发 ping，连续 N 次超时判定死亡。示例策略：

```ts
type HeartbeatPolicy = {
  mode: "poll";
  intervalMs: 15000;    // 正常间隔
  timeoutMs: 5000;      // 单次响应超时
  missThreshold: 3;     // 连续失败次数
  jitterMs: 2000;       // 重连抖动
};
```

关键点：

- 用 `setTimeout` 做单次超时，避免一个慢请求卡住整个探活循环。
- 成功后重置 miss 计数，不要一成功就立即恢复所有状态。
- 判断死亡后进入 `reconnecting`，重连时间加入随机抖动，避免多个实例同时打服务端。

### 3. 推送方案

服务端定时发送 heartbeat，客户端读超时判断。例如 SSE 每 15s 推一次 `:heartbeat\n\n`，客户端如果 35s 没读到任何事件就认为连接异常，主动断开重连。

WebSocket 更简单：协议自带 ping/pong。服务端可以每 20s 发 ping，客户端在 pong 处理器里刷新最后活动时间。客户端也可以反向发 ping，适合浏览器端检测连接是否被网关静默切断。

推送方案的两个常见优化：

- 读超时设为 `interval * 2 + buffer`，不要设得太紧。
- 如果走 SSE 经过代理，心跳帧尽量不用空注释，可以带一个固定事件名，避免被缓冲合并。

### 4. 混合方案

在统一接入层或 MCP 网关中，常见做法是：

- 服务端有业务事件时立即推送；
- 无业务事件时每 15s 推一次 heartbeat；
- 客户端同时保留 45s 的本地轮询兜底。

这样既保证及时，又不依赖某一层连接永远可靠。

## 踩坑点

### 1. 把连接存在当成健康

TCP 连接可以被中间设备保持很久，应用层完全卡死也不会断。HTTP/2、WebSocket 也可能被代理续着。不要用“连接没断”作为健康判断，必须等应用层响应。

### 2. 轮询间隔太短

stdio MCP 每 3s ping 一次，会大量写入 stdin，可能干扰正常 JSON-RPC 消息流。日志噪音和 CPU 抖动也容易盖过真实问题。建议默认 15s 起步，再按场景调整。

### 3. 推送被基础设施静默超时

很多云负载均衡、API Gateway 对空闲连接有 30s 或 60s 超时。heartbeat 间隔必须小于这个 idle timeout，否则推送连接会被默默切断，客户端还误以为健康。可以先用 20s 间隔观察，确认基础设施阈值。

### 4. 单次超时就判死

网络闪断、GC 停顿、系统 sleep 后恢复，都可能造成单次心跳超时。连续失败阈值比单次超时可靠得多。建议至少 `missThreshold >= 2`，并用状态机而不是布尔量。

### 5. 重连风暴

多个 agent 实例同时失去 heartbeat，又同时按固定间隔重连，可能把服务端打挂。重连时间一定要加 jitter，或者用指数退避。

## 可复用建议

- **抽象 HeartbeatPolicy**：把 interval、timeout、missThreshold、jitter 做成配置，底层轮询/推送可替换。不要在每个插件里手写一套。
- **状态机比布尔值好用**：`unknown -> healthy -> suspect -> dead -> reconnecting -> healthy`。状态转换可观测，也能够避免误判后立刻恢复的抖动。
- **记录两个指标**：`heartbeat_loss_count` 和 `heartbeat_rtt_p99`。只看在线/离线不够，心跳延迟升高往往是应用即将卡死的先兆。
- **优先使用协议内置 ping**：MCP JSON-RPC、WebSocket ping/pong、HTTP/2 PING 都优先于自定义文本协议。少造轮子，也更不容易破坏消息边界。
- **和端到端超时分开**：心跳判断节点是否活着，业务调用超时判断一次任务是否失败。两者混用会造成误报。

## 总结

轮询和推送不是技术偏好，而是部署边界决定的。stdio/子进程场景下轮询更稳，WebSocket/SSE 场景下推送更及时，混合方案适合网关和复杂插件宿主。真正重要的是：探活落在应用层、间隔低于基础设施超时、连续失败才判死、重连带 jitter。把这几点做成统一的 HeartbeatPolicy，基本能避开大部分“连接在但已经死了”的问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/063297dc4634e0f5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/0e0565e47c3ac6e7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e6c1c2bd6bb5a409.png)

