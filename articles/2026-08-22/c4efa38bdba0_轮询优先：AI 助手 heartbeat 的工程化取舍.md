---
title: 轮询优先：AI 助手 heartbeat 的工程化取舍
feedId: 34082
source: 综合讨论
publishedAt: 2026-08-22
---

# 背景：heartbeat 在 AI 助手里的真实位置

在 OpenClaw / Agent / MCP / 插件自动化场景里，heartbeat 通常不只是“探活”，它往往承担这些职责：

- 任务执行器向宿主汇报“我还在运行”
- 插件宿主探测 MCP server 是否可用
- 前端 UI 同步 agent 状态（idle / running / error）
- 自动化流程等待外部事件（文件变化、webhook 转发）

这带来两个核心诉求：**及时性**和**稳定性**。

很多开发者一上来就想用 WebSocket 或 SSE 做推送，觉得轮询“不够实时”“有点土”。但真正维护过一段时间后会发现：在不少场景下，轮询反而更省心。这里没有标准答案，只有不同约束下的取舍。

# 问题：长连接不是免费午餐

推送模式（SSE / WebSocket）的优势是低延迟、服务端可主动通知；但代价也很明显：

- 需要维护连接状态、心跳、重连、鉴权刷新
- 在容器 / 反代环境（Nginx、Gateway）里，长连接容易被超时断开
- 客户端数量增加时，服务端连接数会线性膨胀
- 调试困难：抓包、重放、断线复现都比普通 HTTP 请求麻烦

轮询模式的问题则是：

- 有最小延迟：比如 5s 轮询一次，事件发生后最多可能 5s 才被感知
- 高频轮询会放大请求量
- 容易写出“重叠轮询”或“塞满队列”的 bug

所以关键不是哪种技术更先进，而是你的场景对延迟的真实容忍度。

# 做法：以 OpenClaw 插件 / MCP server 为例

## 1. 先定义指标

动手前先问三个问题：

1. 状态变化频率高吗？（每分钟 >1 次？）
2. 客户端需要“服务端主动通知”吗？（例如 token 用尽、外部事件到达）
3. 部署环境是否允许长连接？（是否有反代超时、Serverless 限制）

如果三个答案都是“否”，直接选择轮询。如果任一为“是”，再考虑 SSE / WebSocket，并保留轮询兜底。

## 2. 轮询实现要点

比如一个 OpenClaw 插件需要监控某个任务状态：

```ts
async function pollTask(taskId: string, intervalMs = 5000, signal: AbortSignal) {
  while (!signal.aborted) {
    const status = await fetch(`/api/task/${taskId}`).then(res => res.json());

    if (status.state === 'done' || status.state === 'error') {
      return status;
    }

    await sleep(intervalMs, signal);
  }
}
```

要点：

- 使用 `AbortSignal` / `AbortController` 控制取消
- 上一次请求未返回前，不发起下一次
- 加入 jitter（例如 ±10% 随机偏移），避免多客户端同时打点

## 3. 推送实现要点：SSE 优先

如果确实需要实时，例如 agent 输出流，建议优先用 SSE，而不是裸 WebSocket：

```ts
app.get('/api/task/:id/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const timer = setInterval(() => {
    res.write(`: ping\n\n`); // 注释行保活
  }, 15000);

  req.on('close', () => clearInterval(timer));
});
```

客户端用 `EventSource` 或 `fetch` 流读取。SSE 的好处是天然支持自动重连、事件类型清晰，大多数反代只需配置 `proxy_buffering off` 即可。

## 4. 混合模式：推送加速 + 轮询兜底

工程里最稳的是“推送加速 + 轮询兜底”：

- 正常时：通过 SSE 接收 `state` 事件更新 UI
- 长连接断开或超时：自动降级为每 10–20s 轮询一次
- 重连成功：恢复推送节奏，并用轮询结果做一次全量校准

这样即使推送通道挂了，状态最终仍能一致。

# 踩坑点

1. **轮询风暴**：多个组件各自轮询同一个接口，导致服务端压力。应收敛到宿主统一轮询，再分发给内部订阅者。
2. **定时器未清理**：插件 unload、agent 停止时，不清理 interval / AbortController，会产生泄漏和幽灵请求。
3. **推送的重连风暴**：连接断开后，客户端立即无限重试，服务端一恢复就被打爆。必须加指数退避 + jitter。
4. **把 heartbeat 当任务队列**：heartbeat 只做状态同步，不要用返回体携带大段业务数据，那该走专门的任务结果接口。
5. **忽略时钟偏移**：不要用本地时间判断 expiry，用服务端返回的 `server_time` 或相对 TTL。
6. **SSE 代理缓冲**：Nginx 默认会缓冲，导致“连接建立但收不到数据”。需要 `proxy_buffering off`、`X-Accel-Buffering: no`。

# 可复用建议

- **默认轮询**：能接受秒级延迟的，用 5–15s 轮询，代码简单，排障快。
- **实时优先 SSE**：单向事件流，别一上来就上 WebSocket。
- **必须双向通信**（例如客户端需要发指令）才用 WebSocket，并设计好心跳和断线重连。
- **所有轮询 / 推送都加超时和取消信号**。
- 在日志里区分 `heartbeat_timeout`、`heartbeat_retry`、`heartbeat_failed`，便于观测。
- 做一个运行时配置项，例如 `OPENCLAW_HEARTBEAT_MODE=poll|sse|ws`，切换成本低。

# 总结

heartbeat 设计没有标准答案。OpenClaw 生态里，Agent、MCP server、插件宿主的存活状态通常变化不频繁，轮询足够稳定且易于运维。只有当你明确需要服务端主动推送、且延迟敏感时，再引入 SSE / WebSocket，并且始终保留轮询作为降级路径。

工程上，简单可靠比“看起来实时”更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4683e518e2cca459.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/68a2d92397758a3b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6fd041ef33d74a74.png)

