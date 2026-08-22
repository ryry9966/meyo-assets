---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 34129
source: 综合讨论
publishedAt: 2026-08-22
---

# AI 助手的 heartbeat 设计：轮询 vs 推送的取舍

## 背景
在 OpenClaw 这类 Agent 运行时里，任务经常跨分钟甚至跨小时。一次工具调用可能经过 MCP server，插件宿主也可能长时间不返回。前端或调用方需要一个机制判断“任务还活着吗”“能继续等吗”。这就是 heartbeat 要解决的问题。但 heartbeat 很容易被做成每几秒 ping 一下，实际落地时，轮询和推送的选择会直接影响延迟、负载和误判。

## 问题
OpenClaw/Agent/MCP 场景有几个特点：任务不总在单次 HTTP 请求内完成；工具可能卡住或进程僵死；不同子任务有不同 SLA；自动化任务需要安全取消和状态同步。只用前端轮询 `GET /task/{id}` 简单，但任务多了请求浪费，延迟受轮询间隔限制。只用 WebSocket/SSE 推送及时，但要自己处理保活、重连、背压和网关超时。工程上更合理的是“控制面推送 + 状态面轮询兜底”。

## 做法/步骤
**第一，定义统一 heartbeat 结构。** 不要只发 `{"alive":true}`，至少包含：
```json
{
  "seq": 123,
  "ts": 1710000000,
  "status": "running|blocked|waiting_tool|completed|failed",
  "current_step": "mcp_call:search",
  "timeout_ms": 30000
}
```
`seq` 单调递增，用于识别乱序；`ts` 是服务端时间；`status` 表达当前状态；`timeout_ms` 告诉调用方最晚何时应该有下一次更新，否则可以判死。这样心跳不只是探活，还能驱动前端状态机和超时策略。

**第二，通道规划：控制面推送，状态面轮询兜底。** 任务开始、完成、失败、取消请求等低延迟事件用 SSE 或 WebSocket 推送。SSE 实现简单，适合单向；需要客户端取消指令时选 WebSocket。运行中的进度和步骤更新，可以降低频率，用每 5 秒左右的轮询兜底。只在有活跃任务时开启轮询，任务结束立刻停。

**第三，心跳与业务数据分离。** 推送通道里每 15-30 秒发一个轻量心跳帧，内容保持小于 1KB。大段日志、中间结果走独立事件或轮询接口，避免大 payload 拖慢心跳。

**第四，设置分级超时与重试。** 不同 status 容忍不同超时，`waiting_tool` 可能 30 秒无更新，`running` 10 秒无心跳就告警。客户端超时后不要立即判失败，先触发两次主动探测，两次都失败再标记失联。重连时带上 `last_seq`，让服务端补发事件或当前快照。

**第五，暴露 MCP/插件健康检查。** 让工具或插件实现轻量 `health`，返回自身状态、队列深度和最近错误。Agent 主循环每隔一段时间调用，发现异常就把 `current_step` 标成 `blocked` 上报，而不是继续发 `running`。健康检查要放在独立超时里，比如 2 秒，避免阻塞心跳。

## 踩坑点
1. 网关或 NAT 空闲断开：SSE/WebSocket 云网关可能 60 秒无数据断开，心跳间隔要小于网关空闲超时，不要只靠 TCP keepalive。
2. 推送通道静默失败：连接看着还在，服务端已经不写数据。客户端要按“最后收到心跳时间”判断，超时主动重连。
3. 轮询间隔和任务超时混淆：轮询间隔 5 秒不代表任务 5 秒超时，否则会误杀慢工具。用服务端返回的 `timeout_ms` 判断。
4. seq 不持久化：服务端重启后 seq 从 0 开始，可能把旧连接消息当新消息。用 session_id + 启动时间，seq 只会话内有效。
5. 心跳处理阻塞主循环：同步调用插件 health 可能卡住，拖死整个 agent 心跳，需要异步超时包装。

## 可复用建议
- 心跳结构保持小于 1KB，不要承载业务数据。
- 客户端统一维护 `last_seq`、`last_ts`、`status`、`probe_count`，任何通道更新先过状态机。
- 推送为主，轮询兜底；有活跃任务才开轮询，结束立即停。
- 心跳间隔加 jitter，避免所有客户端同时重连。
- 对 MCP/插件健康检查设置独立超时，降级但不阻塞。
- 监控区分 `heartbeat_timeout`、`probe_failed`、`push_disconnected` 三个指标，排障更快。

## 总结
轮询和推送不是二选一。控制面用推送拿低延迟，状态面用低频轮询做兜底；心跳内容要能驱动超时和取消，而不仅仅是 alive。这样在 OpenClaw/Agent/MCP 多进程、长任务场景下，心跳才能从“能跑”变成“可运维”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7e68b05dd4076c90.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6966e898a68dd42d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/10d2a11c1a38da4d.png)

