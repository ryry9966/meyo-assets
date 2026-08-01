---
title: AI 助手插件的心跳设计：轮询 vs 推送的工程化取舍
feedId: 31196
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

在 OpenClaw 生态中，插件、Agent 或 MCP 服务器往往需要与外部服务保持可感知的连通性——比如对接大模型推理接口、消息队列、长期运行的自动化脚本。这些长连接一旦静默断开，上层 Agent 无法及时感知，就会引发超时、丢任务甚至状态错乱。心跳（heartbeat）就是为此而生：通过定期或事件驱动的方式确认对端存活，必要时触发重连或降级。

实际选型时，开发者通常面临两个方向：**轮询（Polling）** 和 **推送（Push）**。前者用定时器周期性发起健康检查；后者依赖 WebSocket、SSE 或平台事件总线，由服务端主动报告状态。这两个方案没有银弹，但在 OpenClaw 的插件实践中，我们踩出了一些工程化的取舍依据。

## 问题

最直观的纠结在于：

- 轮询实现简单、无状态，适合 HTTP 风格的外部 API，但空闲时也会消耗请求配额和主线程资源；
- 推送模式延迟低、节约带宽，但强依赖底层连接稳定性，实现与排障复杂度明显更高。

更隐蔽的问题是：**心跳不仅是存活探测，还是背压信号、重连计时器和资源释放的触发器**。如果简单粗暴地套用某个模式，很容易掉进“心跳风暴”“幽灵连接”或“事件循环阻塞”的坑里。

## 做法：混合心路历程

下面用一个真实场景拆解：为一个 OpenClaw 插件实现 **“WebSocket 优先 + 轮询降级”** 的心跳模块，该插件负责连接远程模型推理网关，网关同时暴露 WebSocket 和 HTTP 健康端点。

### 1. 核心状态机

在插件代码中定义一个轻量状态机，管理连接生命周期：

```
states: CONNECTING -> CONNECTED -> DISCONNECTED -> RECONNECTING
```

每个状态对应不同的心跳行为，例如 CONNECTED 才启用 WebSocket ping/pong，RECONNECTING 阶段退回轮询检查。

### 2. WebSocket 优先的心跳

使用 `ws` 库建立连接后，借助内置 ping/pong 帧发送心跳：

```javascript
const PING_INTERVAL = 15_000;
let pingTimer;

ws.on('open', () => {
  state = 'CONNECTED';
  pingTimer = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) ws.ping();
  }, PING_INTERVAL);
});

ws.on('pong', () => {
  // 收到 pong，重置超时
  clearTimeout(pongTimeout);
  pongTimeout = setTimeout(() => handleDisconnect(), PING_INTERVAL * 2);
});
```

关键设计：**不在 ping 里直接判定超时，而是用 pong 的缺席触发断线逻辑**，避免网络抖动误判。

### 3. 轮询兜底

当 WebSocket 不可用时（比如某些代理环境、老旧网关），插件会降级为轮询 `GET /health`，间隔比推送心跳更长（如 30 s），并采用指数退避处理失败：

```javascript
async function httpHealthCheck() {
  try {
    await fetch(`${baseURL}/health`, { signal: AbortSignal.timeout(5000) });
    consecutiveFailures = 0;
    interval = BASE_POLL_INTERVAL;
  } catch {
    consecutiveFailures++;
    interval = Math.min(BASE_POLL_INTERVAL * Math.pow(2, consecutiveFailures), MAX_BACKOFF);
    handleDisconnect();
  } finally {
    pollTimer = setTimeout(httpHealthCheck, interval);
  }
}
```

同时，连接恢复后立刻切回 WebSocket 心跳，避免轮询一直占用资源。

### 4. 与 OpenClaw 事件总线集成

插件不是孤立运行的——Agent 或别的插件需要知道这个连接是否健康。可以利用 OpenClaw 内部的 `EventEmitter` 发出状态变更事件：

```javascript
openclaw.emit('plugin:gateway:status', { status: 'disconnected', reason: 'timeout' });
```

这样其他模块可以订阅并做出反应（例如暂停任务入队、激活备用网关）。**推送模式在 OpenClaw 进程内部天然成立**，而对外部服务的探测则交给上面的混合心跳。

## 踩坑点

- **ping interval 过短导致倒灌**：15 s 比较折中，太短（如 5 s）会让某些网关的限流策略误判为恶意行为并断开连接，形成“断连-频繁重连”的抖动。
- **WebSocket 心跳阻塞 Node 事件循环**：务必将 `setInterval` 的 unref 化，否则插件会阻止进程正常退出；我们在插件 `shutdown` 钩子里统一清理所有定时器。
- **轮询和重连风暴**：后退策略如果没做最大间隔限制，故障期间的轮询请求会随指数增大而放大，压垮本地 DNS 或网关。建议设硬上限 60 s。
- **状态不一致**：WebSocket 的 `readyState` 在网络半关闭时可能仍是 `OPEN`，必须用应用层心跳超时兜底。同时注意 `ws` 库在某些版本下 ping/pong 回调时机异常，建议升级到稳定版。
- **OpenClaw 插件的生命周期**：插件禁用或重载时，如果忘记解绑事件和清除定时器，会导致幽灵连接和内存泄漏。请务必将清理逻辑放进 `onDisable()`。

## 可复用建议

结合多次实践，给出一条决策路径：

1. **优先检查外部服务是否原生支持 WebSocket/SSE**，如果支持且客户端网络环境允许（非严格代理、无频繁断网），就采用推送心跳 + 短间隔轮询降级的混合方案。
2. **如果外部是纯 HTTP 服务或无法控制服务端**，直接采用轮询。此时重点优化：使用 `AbortController` 防止挂起、控制并发健康检查数（尤其一个插件管理多个对端时）。
3. **将心跳抽象为独立模块**，提供 `start(transport)` / `stop()` 接口，传入 transport 可以是 `ws` 或 `http`，方便未来切换到 gRPC、SSE 等。模块内部封装状态机、退避算法和事件发布。
4. **在 OpenClaw 内部尽量使用事件总线做同步心跳**，避免让每个插件都去维持自己的 TCP 连接，利用平台已有的连接复用能力。

## 总结

心跳设计看似只是定时 ping 一下，实则牵扯连接生命周期、资源清理、退避策略和事件同步。轮询和推送不是非黑即白的选择，工程上最务实的做法是 **“推送为主、轮询兜底，加上健全的状态机和事件通知”**。这套模式已经被我们在多个 OpenClaw 自动化插件中复用，很少再出现半夜被“僵尸连接”叫醒的情况。

说到底，心跳的终极目的不是证明“我还在”，而是让系统在最短时间内做出正确的恢复动作。把这点想清楚，取舍就顺理成章了。

---

