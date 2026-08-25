---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的工程取舍
feedId: 34758
source: 综合讨论
publishedAt: 2026-08-26
---

# AI 助手的 heartbeat 设计：轮询 vs 推送的取舍

## 背景

在 OpenClaw、Agent、MCP server、插件自动化这类场景里，heartbeat 早已不只是“连接保活”。它承担三类职责：

1. 长任务的存活检测：Agent 执行超过 HTTP 超时或前端等待上限的任务，需要周期性上报进度。
2. MCP server / 插件进程的健康上报：注册中心要靠心跳判断一个 tool/plugin 是否在线。
3. 事件与状态同步：某个工具完成、外部回调到达、任务失败，这些变化需要尽快被消费方感知。

因此，heartbeat 设计会直接影响系统的延迟、可靠性以及运维成本。

## 问题：轮询还是推送

轮询的典型做法是客户端每隔 N 秒请求一次 `/health` 或 `/events?since=seq`。优点是实现简单、跨网络穿透好、服务端基本无状态；缺点是空转开销大、实时性受间隔限制，间隔太小容易触发网关限流或 serverless 计费问题。

推送通常用 SSE、WebSocket 或 webhook。优点是低延迟、服务端主动通知；缺点是连接管理复杂，要处理鉴权、重连、代理超时、半开连接和背压。

工程上很少二选一，更常见的是按“实时性要求 vs 运维复杂度”分层：控制链路可以推送，状态查询可以轮询；推送断线后降级轮询。

## 做法 / 步骤

可以按下面顺序落地一套比较稳的 heartbeat。

### 1. 先定义消息结构

不要只返回一个 `ok`。建议心跳包至少包含：

```json
{
  "schema_version": 1,
  "seq": 202,
  "ts": 1710000000,
  "status": "running",
  "progress": 0.62
}
```

`seq` 是事件游标的核心。轮询用 `since=seq` 增量拉取，推送重连后用 `seq` 恢复，避免重复消费或漏事件。

### 2. 设定三层阈值

- `heartbeat_interval`：10–30 秒，按任务类型可配。
- `timeout`：2–3 个 interval，超过则标记 `stale`。
- `retry`：指数退避，例如 1s、2s、4s、8s，最大 30s。

不建议一超时就立刻 kill。给一个 grace period，先标记 `stale`，再进入 `unhealthy`，最后才回收资源。

### 3. 轮询实现

客户端保存 `last_seq`，请求：

```
GET /events?since=last_seq&limit=50
```

服务端只返回 `seq > last_seq` 的新事件；没有新事件也返回一个心跳包。服务端用环形缓冲或数据库游标保存最近事件，避免客户端长时间离线后一次性补拉几万条。

### 4. 推送实现

SSE 适合单向事件流，WebSocket 适合需要双向控制的场景。连接建立后服务端先发一个 snapshot，再发 delta。客户端在断线重连时带上 `Last-Event-ID` 或自定义 header，服务端据此恢复。

### 5. 混合策略

推送断线后自动降级轮询，并把最后收到的 `seq` 传给轮询接口。这样切换过程中不丢事件。

### 6. Agent / 插件侧接入

MCP server 启动时向 registry 发一次 hello，之后周期上报。registry 端只根据 heartbeat 的 `seq` 和 `ts` 判断新鲜度，不在心跳里塞大字段。

## 踩坑点

这些是实际工程中比较容易踩到的：

- **轮询间隔太短导致限流**：尤其是云函数、API Gateway 按请求计费时，10 个 agent 每 5 秒轮询一次，量级会很快上去。
- **NAT / 代理静默断开**：不少云环境会在 60–90 秒断开空闲连接。SSE/WebSocket 只靠 TCP keepalive 不够，必须在应用层发心跳帧。
- **心跳与业务事件混在一起**：重连时容易重复消费。必须用 `seq` 或事件 ID 做幂等。
- **移动端 / 笔记本休眠**：恢复后一次性拉取大量事件。限制 `limit`，并让客户端用游标分页补齐。
- **服务端重启后内存 seq 归零**：客户端旧 `seq` 越界会触发全量拉取或错误。建议 seq 持久化，或加 epoch 前缀。
- **只判断连接存在，不判断任务进度**：进程活着不代表任务在推进。heartbeat 应携带 `progress` 或 `status`，否则 agent 卡死仍显示在线。

## 可复用建议

如果你的 OpenClaw / Agent / MCP 项目要重新设计 heartbeat，可以参考这几点：

1. 固定 `schema_version` 和 `seq`，避免“心跳能通但字段不兼容”。
2. 推送和轮询共用同一套事件游标模型，不要把轮询做成全量查询。
3. 心跳间隔做成环境变量或配置文件项，不要写死。
4. 监控 heartbeat 延迟、重连次数、漏事件数，而不是只看在线/离线。
5. 把 heartbeat 做成一个独立 transport 层，业务只依赖 `on_event` / `on_status`，不关心底层是轮询还是推送。

## 总结

heartbeat 设计本质是状态同步的成本与延迟权衡。单靠轮询简单但滞后，单靠推送低延迟但脆弱。工程上更稳妥的是：

> 推送优先、轮询兜底、seq 对齐、幂等消费。

在 agent 场景里，心跳还必须携带任务状态和进度，否则它只是一个昂贵的“我还活着”信号，并没有业务价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/c3344cb0300208c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/13b8e100ce138ba0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/82974963d61c3d97.png)

