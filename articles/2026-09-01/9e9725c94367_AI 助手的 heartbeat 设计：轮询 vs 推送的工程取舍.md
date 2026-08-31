---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的工程取舍
feedId: 35619
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw、Agent、MCP 插件、自动化任务等场景中，heartbeat（心跳）几乎是必备的基础设施。它被用来做健康检查、任务状态同步、连接保活、配置热更新等。但很多实现要么轮询过于频繁打爆服务端，要么推送连接一断就完全失去感知。本文结合实践，聊聊轮询与推送的取舍，以及如何设计一个可靠的 heartbeat 机制。

## 问题

轮询实现简单、防火墙友好，但存在资源浪费和延迟问题；推送实时性好、服务端可主动通知，但引入了连接管理、断线重连、背压等复杂度。实际工程中，我们很少能只选一种，更多是在不同场景下做权衡，甚至混合使用。

## 做法/步骤

### 1. 明确心跳目的

先问自己：心跳是为了检测进程存活？还是为了通知任务进度？或者只是维持长连接不被代理断开？不同目的对实时性、可靠性、负载的要求完全不同。例如存活检测可以容忍 30 秒延迟，但任务状态通知往往需要秒级甚至亚秒级。

### 2. 分离心跳通道与业务数据通道

心跳包应该足够小，只携带最小必要信息：版本号、时间戳、状态码、最后事件 ID。业务数据不要混在心跳里，否则心跳包变大，序列化/反序列化开销增加，也容易在重试时重复处理业务逻辑。

### 3. 轮询实现要点

如果选择轮询，建议遵循以下模式：

```typescript
async function pollLoop() {
  let lastEventId = 0;
  while (running) {
    try {
      const res = await fetch(`/events?since=${lastEventId}`);
      if (res.ok) {
        const events = await res.json();
        for (const ev of events) {
          handle(ev);
          lastEventId = ev.id;
        }
      }
    } catch (err) {
      // 记录错误，按退避策略重试
    }
    await sleep(interval + jitter());
  }
}
```

关键点：

- 使用固定间隔 + 随机抖动（jitter），避免所有客户端同时请求造成惊群。
- 通过 `since` 参数拉取增量事件，避免重复拉取。
- 设置超时和最大重试次数，防止请求堆积拖垮客户端。

### 4. 推送实现要点

推送优先考虑 WebSocket 或 SSE（Server-Sent Events）。Webhook 适合服务端到服务端的单向通知，但需要公网可达，且不够实时。

WebSocket 示例：

```javascript
function connect() {
  const ws = new WebSocket(url);
  ws.on('open', () => {
    // 发送订阅或鉴权信息
    ws.send(JSON.stringify({ type: 'subscribe', topic: 'task-events' }));
  });
  ws.on('message', (data) => {
    const msg = JSON.parse(data);
    if (msg.type === 'heartbeat') {
      // 更新最后心跳时间
      lastHeartbeat = Date.now();
    } else {
      handle(msg);
    }
  });
  ws.on('close', () => {
    const delay = Math.min(maxDelay, baseDelay * 2 ** attempts) + jitter();
    attempts++;
    setTimeout(connect, delay);
  });
}
```

关键点：

- 应用层心跳保活：即使底层 WebSocket 有 ping/pong，也要在应用层定期发送心跳，防止代理/NAT 静默断开。
- 断线重连采用指数退避 + 随机抖动，避免服务端重启后大量客户端同时重连造成雪崩。
- 服务端维护连接注册表，处理半开连接和僵尸连接。

### 5. 混合策略

推荐“推送为主、轮询兜底”的模式。例如正常情况通过 WebSocket 推送事件通知，同时客户端每 30 秒轮询一次健康检查接口，确认服务端存活。一旦推送断开，立即切换到高频轮询（如每 5 秒），直到推送恢复。

## 踩坑点

- **轮询间隔过短**：导致数据库或接口压力剧增，尤其是多个客户端同时轮询时。建议通过监控和压测确定合理间隔。
- **推送连接被静默断开**：生产环境常有负载均衡、反向代理，它们的空闲超时可能小于心跳间隔，导致连接被回收。必须确保应用层心跳间隔小于所有中间设备的最小空闲超时。
- **重试风暴**：服务端重启后，大量客户端同时重连，可能再次打垮服务端。客户端必须加随机延迟，服务端要做限流和优雅启动。
- **心跳与业务状态混淆**：心跳成功只代表连接存活，不代表任务处理成功。业务状态需要单独的确认机制，不能依赖心跳。
- **消息乱序**：推送消息可能乱序到达，需要带单调递增的序列号或时间戳，客户端做排序或去重。

## 可复用建议

1. **抽象 Heartbeat 接口**：定义统一的 `start()`、`stop()`、`onEvent()`、`onDisconnect()` 等方法，底层可以替换为轮询、WebSocket、SSE 等传输，业务代码不感知。
2. **配置化**：心跳间隔、超时、重试次数、退避系数都做成配置项，方便在不同环境（本地、测试、生产）调优。
3. **监控指标**：至少采集心跳成功率、平均延迟、重连次数、队列长度。这些指标能提前暴露问题，比事后排查高效得多。
4. **复用成熟库**：MCP SDK 自带心跳机制，WebSocket 库通常有自动重连，SSE 有成熟的事件解析库。不要重复造轮子。
5. **注意环境差异**：本地直连没问题，不代表生产没问题。检查负载均衡、反向代理的空闲超时设置，确保大于心跳间隔。

## 总结

轮询适合低频、简单、防火墙友好、不需要实时推送的场景；推送适合高频、实时、服务端需要主动通知的场景。工程实践中，我更倾向于“推送为主、轮询兜底”的混合模式，并做好连接管理、退避重试和监控。心跳设计本身不难，难的是在资源消耗、感知延迟、系统可靠性之间找到平衡。希望这些经验能帮你在 OpenClaw 或 Agent 项目中少走弯路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/596df482e4e6a4b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/f08d74c92b85a80e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b3113534e2e81723.png)

