---
title: AI 助手 heartbeat 设计：轮询 vs 推送，别把简单问题做成分布式系统
feedId: 32957
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

OpenClaw 这类 Agent 项目跑起来后，很快会遇到一个现实问题：异步任务结束了怎么通知？插件状态是否还活着？MCP server 有没有新事件？这些需求最终都会落到同一个设计上——heartbeat。

很多人一上来就上 WebSocket，结果被断线重连、网关超时、消息重复搞得疲惫。实际上，大多数自动化场景里，轮询已经足够。Heartbeat 设计的关键不是协议选型，而是先定义清楚三个变量：**事件频率、延迟容忍、连接规模**。

## 问题

heartbeat 不该只做“存活检测”，它本质是一个轻量事件通道。不同场景对它的要求差异很大：

- 定时巡检、任务状态变更：分钟级，延迟容忍高，轮询完全够用。
- 插件通知、MCP 工具结果回传：秒级，单向通知，SSE 或长轮询更合适。
- 实时交互、双向控制：高频、低延迟，才值得上 WebSocket 或 MQTT。

如果你在一个跑在用户机器上的 OpenClaw 插件里，网络可能还要穿透公司网关或反向代理。这时 HTTP/SSE 的部署成本通常远低于 WebSocket。

## 做法 / 步骤

以 OpenClaw Agent + MCP + 本地插件为例，建议按下面的顺序落地。

**1. 先把 heartbeat 落到状态表，而不是只发信号**

建议维护一张 `heartbeat(agent_id, kind, last_seen_at, seq, payload)`。每次更新时写入 `last_seen_at`，有事件时递增 `seq`。这样无论后面用轮询还是推送，都有持久化基线。

**2. 默认先做轮询**

Agent 每 8~15 秒查一次：

```
GET /api/heartbeats?since_seq=<last_seq>
```

服务端只返回增量。查询走 `(agent_id, seq)` 索引，响应加 `Cache-Control: no-store`。间隔不要拍脑袋设成 1 秒，先问自己是否真的需要亚秒级延迟。

**3. 需要秒级通知时，再加 SSE**

在已有 HTTP 服务上加一个流式端点：

```
GET /api/heartbeats/stream
```

客户端保存 `seq` 或 `Last-Event-ID`，重连时从上次游标继续。反向代理要关闭缓冲：

```nginx
proxy_buffering off;
proxy_set_header X-Accel-Buffering no;
```

否则你会看到事件明明发出去了，客户端几十秒后才收到。

**4. 本地插件/MCP 优先走进程内或本地通道**

如果 MCP server 和 Agent 在同一台机器上，可以考虑直接读本地 SQLite、Unix socket，甚至让 MCP server 暴露一个 `read_heartbeat` 工具，由 Agent 主动调用。这本质上是“按需轮询”，少一层网络，少一类故障。

**5. 只有真正双向高频控制时才上 WebSocket**

如果确实需要 WebSocket，应用层心跳必须有，15~30 秒一次 ping/pong，连续 2 次未响应就重连。重连要加指数退避和随机抖动，避免服务端重启后的重连风暴。

## 踩坑点

- **轮询 QPS 失控**：多个 Agent 同时 1 秒轮询，数据库连接很快打满。给轮询间隔加 jitter，比如随机 5~10 秒，避免同步冲击。
- **SSE 被网关缓冲**：表现为事件延迟几十秒，排查半天发现是反向代理在缓冲。先查 `X-Accel-Buffering` 和 `proxy_buffering`。
- **把推送当数据源**：推送通道天然会丢消息、重复、乱序。收到通知后，应该再拉一次完整状态，推送只做“唤醒信号”。
- **重连风暴**：服务端一重启，所有客户端同时重连，可能直接把服务打挂。重连必须加 jitter 和最大退避时间。
- **只用 TCP keepalive 判断存活**：中间 NAT 设备会静默断开空闲连接。应用层心跳才是唯一可靠的存活判据。
- **MCP/插件进程假活**：心跳正常并不代表逻辑没卡死。建议在 heartbeat payload 里带上当前任务 phase 或队列深度，让心跳不只是“我还活着”。

## 可复用建议

- **先轮询，后推送，不要一上来就 WebSocket。**
- **所有事件接口统一带 `seq` 游标**，保证可重放、可续传。
- **推送通道必须可降级**：SSE 连不上时，客户端自动退回轮询。
- **状态以持久化为准，通知只是通知**。数据库/状态表是唯一事实来源。
- **给 heartbeat 加可观测性**：记录 `last_heartbeat_age`、`missed_count`、`stream_reconnects`，否则出问题时你只能靠猜。

## 总结

heartbeat 设计的目标，是让 Agent 以最小成本知道“世界发生了什么变化”。多数自动化场景下，8~15 秒的轮询已经足够；需要秒级单向通知时，SSE 是比 WebSocket 更务实的选择；只有真正需要双向高频控制时，才值得引入 WebSocket。

先确定频率和延迟容忍，再选择通道。不要反着来。

---

