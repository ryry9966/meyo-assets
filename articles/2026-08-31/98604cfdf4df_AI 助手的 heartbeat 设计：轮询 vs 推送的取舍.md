---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 35531
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw/Agent 项目里，heartbeat 通常分为控制面存活检查与任务面进度确认。很多实现直接沿用轮询：每 N 秒请求 `/healthz` 或查 Redis。节点少时没感觉，一旦挂上十几个 MCP server、浏览器 Agent、沙箱执行器，固定轮询会放大延迟、日志和误报。推送看似更实时，但连接管理、重连、背压、乱序都会变成新问题。

本文给出一个可落地的分层方案：控制面轮询、任务面租约、事件面按需推送。

## 问题

轮询最大的坑是“假活”：进程在但 event loop 被长任务阻塞，`/healthz` 仍可能返回 200。推送最大的坑是“假死”：broker 或客户端断线，健康状态没有及时收敛。

常见错误：

- 全局只用一个心跳，不区分进程存活与任务卡死。
- 心跳过密，指标和日志爆炸。
- 把心跳事件当业务事件源，状态机被乱序消息打乱。
- MCP 插件挂 HTTP 长轮询但不设取消，任务结束请求还挂着。

## 做法

1. **控制面**：15-30s 轮询 `/healthz`，返回版本、最近成功任务时间。仅用于发现进程退出、网络隔离。
2. **任务面**：超过 5s 的任务必须注册租约，例如 Redis key `lease:task:{id}` TTL 10s，任务每 5s 续约。调度端发现租约过期则标记 stalled，可重试或降级。
3. **事件面**：进度、日志、工具结果用 SSE/WebSocket 推送；但事件只做展示和轻量回调，不作为唯一状态源。
4. **实现细节**：租约不要用 `SET NX`，直接 `SET EX` 覆盖即可；续约要独立线程，避免主循环阻塞。

## 踩坑点

- `/healthz` 的 HTTP client 未设超时，慢请求占满线程池。健康检查应 2-3s 超时。
- 推送通道与业务回调共用 consumer，慢任务阻塞心跳分发。
- 长任务被 `kill -9` 后，租约过期才被发现；TTL 建议为 2-3 个续约周期。
- SSE/WS 空闲连接被 NAT/代理断开，必须发心跳帧。
- Webhook 重复事件要按 `task_id + event_seq` 幂等去重。

## 可复用建议

- 不要一刀切：控制面轮询、任务面租约、事件面推送。
- 延迟能接受 5-10s 时，优先 Redis 轮询或 PG `LISTEN/NOTIFY`，少维护长连接。
- 心跳消息带 `seq` 和 `timestamp`，便于观察乱序和延迟。
- 做一个 `HeartbeatPolicy`：`mode=poll|lease|sse|ws`，插件可按需覆盖。
- OpenClaw/Agent 模板里先实现租约续约，再考虑推送；多数失败不是进程退出，而是任务卡住。

## 总结

轮询和推送不是二选一，而是按层分工。控制面用轮询保证可发现，任务面用租约保证可恢复，事件面用推送优化体验。低频轮询加带 TTL 的任务租约，通常比复杂推送基建更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2010be59ae265a06.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f597ce6c62309d83.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8ff3bf6c4d7db666.png)

