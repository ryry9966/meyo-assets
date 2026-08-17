---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 33598
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 的 Agent 编排里，助手很少是孤立运行的：它可能等一个 MCP 工具返回、监听插件事件、跟自动化任务队列同步，或者确认远端 worker 是否还活着。heartbeat 不只是“对方在线吗”，它通常同时承担四件事：

- **liveness**：进程 / 会话是否存活
- **progress**：长任务进展到哪了
- **event**：外部状态发生变化
- **cancellation**：取消、暂停、降级信号

工程上实现 heartbeat 通常只有两条路：**轮询 polling** 和 **推送 push**，后者包括 SSE、WebSocket、webhook、MCP notifications 等。

## 问题

轮询的优点是无状态、容易穿透防火墙、状态可重放；缺点是间隔短了会打爆接口，间隔长了延迟又大。

推送的优点是低延迟、省资源；缺点是连接管理复杂：断线重连、代理超时、背压、事件丢失、客户端假死都很常见。

真实项目里一般不是二选一，而是把 heartbeat 按目的拆开，不同层用不同策略。

## 做法与步骤

我们在一套 OpenClaw 插件 + MCP 连接器里用了这样的分层方式：

1. **先分类 heartbeat 目标**  
   在配置文件里明确每个 heartbeat 的用途，不要一个接口又当存活检测，又当进度查询。

2. **Liveness 保持轻量**  
   存活检测用短周期轮询，比如 5–15 秒一次，并加 jitter。接口只返回时间戳、版本号或序列号，不做全量状态序列化。进程内部也可以直接定时上报，避免额外 HTTP 请求。

3. **Progress 用版本化轮询或主动推送**  
   长任务内部主动更新状态文件或 progressToken；消费端用 1–3 秒的轮询拉取，但带上 `If-None-Match` 或版本号，只有变化时才真正解析。不要每次都生成完整任务树。

4. **Event 优先推送，但必须有补偿轮询**  
   SSE / WebSocket / MCP notifications 适合事件驱动。但推送链路可能断，所以每 30–60 秒用 seq / cursor 拉一次增量，把断线期间漏掉的事件补齐。

5. **连接管理要有应用层心跳**  
   推送连接空闲 15–30 秒发一次 ping 或 comment，设置 read/write deadline。重连用指数退避，上限 30–60 秒，并加随机 jitter，避免所有客户端同时重连。

6. **所有等待 heartbeat 的地方挂 timeout**  
   在 Go 里我们是 `context.WithTimeout`，在其他语言里也应该有等效的 deadline。连续丢失 N 次心跳后触发降级或终止，而不是无限等下去。

## 踩坑点

- **把轮询做成“准实时”**  
  曾经把 poll interval 设成 200ms，结果 MCP server 触发 rate limit，远端 API 开始拒绝请求，本地 CPU 也明显升高。实时感应该靠推送解决，轮询只做兜底。

- **推送连接假死**  
  NAT 或代理把空闲连接静默断开，客户端没收到 FIN，一直以为连接正常。没有应用层 ping 就会卡住很久。

- **只依赖推送，没有 seq 补偿**  
  断线期间的 event 丢失后，状态会漂移。后来所有 event 都带递增 seq，客户端重连后按 seq 拉增量。

- **heartbeat 接口做重活**  
  有个版本把任务树全量 JSON 每次都返回，几百个任务时心跳接口自己成了瓶颈。后来的接口只返回变化的版本号。

- **多个组件各自独立轮询**  
  插件、Agent、UI 同时轮询同一个 endpoint，造成 thundering herd。统一到一处 poll，再由内部广播，或者加 jitter 和共享缓存。

## 可复用建议

- **定义统一 Heartbeat 接口**  
  建议至少包含三个方法：`Ping()`、`State()`、`Watch(seq)`。内部决定走轮询还是推送，外层不关心协议。

- **MCP 工具优先用标准通知机制**  
  长任务用 progressToken 推送进度，事件变化用 MCP notifications；消费端仍然设置轮询兜底，别把推送当 100% 可靠。

- **本地插件 / 自动化优先用文件 watcher 或 Unix socket**  
  跨机器再考虑 WebSocket / SSE。本地文件 watcher 比轮询更实时，比网络推送更省心。

- **配置化而不是硬编码**  
  把 poll interval、push retry、timeout、max reconnect backoff 写进 yaml，针对不同 connector 可单独调整。

- **可观测性**  
  记录心跳间隔、丢包 / 重连次数、端到端延迟，而不是只看“是否在线”。很多故障是心跳质量变差，而不是完全断开。

## 总结

轮询不是原罪，推送也不是银弹。合理的做法是：**存活检测轻量轮询，进度同步按需拉取，事件变化推送 + 补偿轮询，所有等待都要有 timeout 和降级路径。**

先分清 heartbeat 的用途，再决定轮询还是推送，比纠结某一种协议更重要。

---

