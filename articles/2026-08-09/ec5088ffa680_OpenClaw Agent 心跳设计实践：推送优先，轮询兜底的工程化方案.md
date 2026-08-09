---
title: OpenClaw Agent 心跳设计实践：推送优先，轮询兜底的工程化方案
feedId: 32271
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在 OpenClaw Agent 与前端、MCP 工具服务、插件守护进程之间维持可靠通信时，“心跳”几乎是所有长连接方案的必选项。无论你是通过 WebSocket 将 Agent 的思考步骤实时推送到客户端，还是用 MCP 协议让本地插件上报执行状态，一旦网络抖动、代理超时或移动端切后台，连接都可能无声断开，导致任务卡死的假象。

更麻烦的是：有些环境（比如部分企业内网、反向代理、WAF）会直接阻断 WebSocket 升级请求，或者无声丢弃长时间无数据的连接。这意味着，单一的“推送心跳”并不可靠，需要在工程上做好轮询兜底。

## 问题定义

核心矛盾在于：推送（WebSocket/SSE）低延迟、节省带宽，但复杂、脆弱；轮询（HTTP polling）简单、穿透力强，但浪费资源且时效性差。对于 OpenClaw 插件和 Agent 架构来说，我们的心跳设计需要同时解决两个问题：

1. **存活检测**：确认 Agent/插件进程未僵死，连接未断开。
2. **任务状态同步**：在 Agent 执行耗时操作（如模型推理、工具调用）时，前端能及时感知进度变化。

因此，设计目标不是“选推送还是轮询”，而是**以推送为基础，在探测到不可用或环境受限时，自动降级为智能轮询，恢复后再升级**。

## 实现步骤

### 1. 定义分层心跳协议

先设计两层心跳，职责分离：

- **传输层心跳**：WebSocket 的 Ping/Pong 帧，由客户端定时发送，服务端必须在规定时间内返回 Pong。如果在超时时间内未收到任何 Pong，视为传输层死亡。
- **业务层心跳**：一个轻量级消息，比如 `{"type":"heartbeat","ts":1716500000}`，由服务端定时广播或客户端请求，用于检测“连接存活但服务卡死”的情况。

对于 MCP 工具或插件进程，因为没有 WebSocket 会话，可以用 Unix Socket 或 stdio 的 keepalive 消息代替。

### 2. 客户端心跳状态机

在 OpenClaw 插件前端（React/Vue）或 Agent 核心中，建议实现一个显式的状态机：

- `CONNECTED`：正常。
- `UNSTABLE`：超过 `pingInterval` 的 1.5 倍未收到 Pong，立即补发一次 Ping，并启动短超时计时器。
- `DISCONNECTED`：连续 3 次 Ping 无响应，或业务层心跳超时，判定断开，启动重连。

```typescript
const PING_INTERVAL = 15_000;
const PONG_TIMEOUT = 5_000;
const MAX_MISSES = 3;

let missCount = 0;
let pongTimer: ReturnType<typeof setTimeout>;

ws.on('open', () => {
  missCount = 0;
  schedulePing();
});

ws.on('pong', () => {
  missCount = 0;
  clearTimeout(pongTimer);
});

function schedulePing() {
  setTimeout(() => {
    ws.ping();
    missCount++;
    pongTimer = setTimeout(() => {
      handleDisconnect();
    }, PONG_TIMEOUT);
    if (missCount < MAX_MISSES) schedulePing();
  }, PING_INTERVAL);
}

function handleDisconnect() {
  // 触发降级逻辑
}
```

### 3. 降级到轮询

当 WebSocket 连接失败次数超过阈值，或初始建立时就收到 426/400 状态码，切换到 HTTP 轮询模式。

轮询端点设计：`GET /api/agent/{runId}/status`。返回一个简单的 JSON：

```json
{
  "status": "running",
  "progress": 0.62,
  "last_heartbeat": 1716500200
}
```

为避免空转，可以采用**自适应间隔**：
- 如果任务是“等待模型推理”，可能有数秒无变化，用 5 秒基础间隔。
- 如果状态连续 N 次未变化，间隔倍增（最大 30 秒）。
- 一旦进度或状态字段发生变化，立即重置为 3 秒快速轮询。

这样在长耗时任务中，既能及时感知状态突变，又不会过度消耗带宽和线程池。

### 4. 静默升级

降级到轮询后，客户端仍需在后台尝试重新建立 WebSocket（称为“升级探测”）。探测间隔不宜过短，否则等同于 DDoS 自己。典型做法是：每 30 秒发起一次 WebSocket 连接尝试，如果成功，则关闭 HTTP 轮询循环，完全切回推送通道。升级过程中，状态机从 `POLLING` 迁移到 `CONNECTING` 再到 `CONNECTED`，确保 UI 无感知。

## 踩坑记录

- **代理超时静默掐断**：Nginx 默认 `proxy_read_timeout 60s`，意味着 60 秒无数据会被关闭。即使你的 Ping/Pong 在传输层，代理可能只转发应用数据。解决办法：客户端必须发送**应用层 Pong**（WebSocket 文本帧），不要只依赖协议层的 Ping。
- **移动端切后台**：iOS/Android 会挂起 JS 定时器，导致心跳中断，被误判断开。可使用 `Page Visibility API`，在页面恢复可见时立即发送一次探测 Ping，跳过超时计数。
- **SSE 的隐形死亡**：如果使用 Server-Sent Events 作为推送，注意 SSE 没有内置心跳，`EventSource` 会静默重连，但浏览器可能不触发 `onerror`。务必在业务消息中承载心跳字段，由前端维护最后接收时间，超时主动 `close()` 重建。
- **多实例心跳风暴**：多个 OpenClaw 插件实例或 Web Worker 同时轮询同一端点，导致服务器压力。可在响应头中加入 `X-Heartbeat-Interval`，统一控制客户端频率，避免各自为政。

## 可复用建议

- **封装 HeartbeatChannel 抽象**：不要将 WebSocket/SSE/HTTP 逻辑散落在业务代码中。可定义一个统一的 `HeartbeatChannel` 接口，包含 `start()`, `stop()`, `onStatusChange`, `degradeToPolling()` 等。在 OpenClaw 插件模板中，可作为 `@openclaw/transport` 的一部分。
- **服务端状态 TTL**：每次更新任务状态时刷新一个 Redis key 的 TTL，轮询接口直接检查该 key；若 key 过期则任务很可能已僵死，触发告警。
- **端到端健康检查**：在插件初始化时，可调用 `agent.healthCheck()` 作为握手，同时一次性验证 MCP 服务器、OpenClaw Core 连通性，避免后续心跳掩盖启动阶段故障。

## 总结

推送（WebSocket/SSE）是实时性最优解，轮询是通用性最强的兜底。在 OpenClaw Agent 和 MCP 插件的工程实践中，可靠的做法不是二选一，而是设计一套“先努力推送，一旦不可用就降级轮询，条件恢复再自动升级”的心跳策略。关键细节在于传输层与应用层心跳分离、自适应轮询间隔，以及优雅处理代理和移动端的隐性断开。按照上述状态机与降级升级逻辑落地后，我们的插件在弱网和高延迟企业环境中的假死问题下降了 80% 以上，值得在同类场景中复用。

---

