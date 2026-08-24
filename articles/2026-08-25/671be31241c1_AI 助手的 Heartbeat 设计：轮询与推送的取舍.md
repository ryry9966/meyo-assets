---
title: AI 助手的 Heartbeat 设计：轮询与推送的取舍
feedId: 34609
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent/MCP/插件自动化场景里，heartbeat 不只是“进程还活着”的 ping。它通常承载三类信息：

- liveness：MCP server、插件进程是否存活；
- 状态同步：任务是否还在运行、进度如何；
- 完成通知：异步任务是否已经结束。

heartbeat 设计会直接影响延迟、资源占用和排障难度。很多自动化链路里，Agent 需要根据 heartbeat 判断是否重试、降级或切换实例。

## 问题

纯轮询实现简单，客户端主动发起，天然适合本地开发和 NAT 后环境。但轮询的延迟与开销成正比：间隔太短，MCP server 会被大量无效请求打满；间隔太长，任务完成几分钟后 Agent 才感知到。

纯推送延迟低，但需要长连接或公网回调。断线重连、鉴权、乱序、重复消费都是成本。OpenClaw 自动化中大量插件运行在本地或内网，不能假设公网 webhook 一定可达。

所以真正的问题不是“选轮询还是推送”，而是：在有限的资源下，如何用较低复杂度拿到可接受的实时性和可靠性。

## 做法/步骤

### 1. 先分类，再选通道

不要用同一种 heartbeat 承载所有信息。建议拆成：

- liveness：低频轮询即可；
- 状态变更：走推送；
- 任务队列扫描：保留定时轮询兜底。

### 2. 推拉结合

插件或 Agent 可以暴露 SSE 或 WebSocket，关键事件即时推送；同时保留 30-60s 的轮询兜底，防止推送断线后漏掉状态。推送负责“快”，轮询负责“保底”。

### 3. 轮询加游标

不要每次都拉全量。使用 `since_id`、`If-Modified-Since` 或 resource 版本号，只获取增量变化。否则 heartbeat 本身就是一种数据放大。

### 4. 心跳载荷最小化

heartbeat 载荷只保留：

- 实例 ID；
- 版本号；
- 租约时间；
- 状态码；
- 最后事件序列号。

业务数据不要塞进 heartbeat，应该通过事件通道单独拉取。

### 5. 健康判定要容忍抖动

连续失败 2-3 次才标记 unhealthy，恢复后重置计数。用“最后成功时间 + 连续失败次数”判断，而不是单次超时直接判死。

### 6. 推送侧做幂等和版本控制

事件带 `event_id` / `seq`，消费端按 ID 去重；状态更新带 `version`，拒绝旧版本覆盖新版本。

一个简单兜底逻辑如下：

```text
loop every 45s:
    resp = GET /mcp/status?since=last_seq
    if timeout or status != 200:
        fail_streak += 1
    else:
        last_seq = resp.last_seq
        fail_streak = 0
    healthy = fail_streak < 3

on event push:
    if event.seq <= last_seq: ignore
    apply(event)
    last_seq = event.seq
```

## 踩坑点

- **把 heartbeat 当实时通知用**：轮询间隔设成 1s，MCP server 的 CPU 和网络被无效请求消耗，实际任务处理能力反而下降。
- **推送断开后没有兜底扫描**：事件丢失几小时无人发现，Agent 一直以为任务还在运行。
- **多实例同时轮询同一队列**：导致重复执行。需要 single leader、分布式锁，或者用原子操作领取任务。
- **心跳日志未采样**：5s 写一次库，磁盘膨胀，排障时被噪声淹没。
- **本地 webhook 开发不可达**：直接上公网回调调试困难。应优先使用 WebSocket/SSE 这类从内向外的长连接。
- **推送乱序覆盖新状态**：网络重传导致旧状态覆盖新状态，必须有 sequence/version 判断。

## 可复用建议

- 探活间隔 30-60s，任务状态 5-15s，交互式操作走推送。
- 即使有推送，也保留一条低频轮询路径作为 fallback。
- 将 heartbeat 状态暴露给 OpenClaw Agent 作为可观测信号，让模型根据连续失败次数决定是否重试或降级。
- 记录最小元数据：`last_seen`、`fail_streak`、`source_instance`、`lease_expire_at`。
- 对本地插件，优先选择出站长连接，而不是公网 webhook，减少网络配置成本。

## 总结

Heartbeat 不是越频繁越好，也不是有了推送就不要轮询。工程上比较稳的做法是：低频轮询保底、事件推送提速、心跳载荷最小化、消费端幂等。

把 heartbeat 设计成可观测、可恢复的状态信号，而不是单纯的 ping-pong，才能让 Agent、MCP server 和插件在长时间自动化任务里稳定协作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c20e6b87fadd431e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/aeba99cbe049a05a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d04abb1190998650.png)

