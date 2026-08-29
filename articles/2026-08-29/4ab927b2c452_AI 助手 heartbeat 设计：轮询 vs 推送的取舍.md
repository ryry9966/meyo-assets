---
title: AI 助手 heartbeat 设计：轮询 vs 推送的取舍
feedId: 35243
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent、MCP 工具与插件自动化中，heartbeat 通常承担三类职责：保活、状态同步、异常发现。比如一个自动化流程调用 MCP 工具后等待外部审批，或者 Agent 轮询子任务进度，都需要一个“心跳”来确定对端是否还活着、状态是否变化、结果是否到达。

问题不在于“轮询还是推送更好”，而在于：不同通道的延迟、资源消耗、实现复杂度完全不同。轮询实现简单、绕过防火墙容易，但无效请求多，延迟受间隔限制；推送（SSE/WebSocket/Webhook）实时性高，但连接保持、断线重连、顺序、去重、鉴权都会成为问题。

## 做法/步骤

**1. 先定义 heartbeat 的目标**

不要一上来就写定时器。先回答：这个心跳是为了保活连接、同步任务状态，还是检测任务是否超时？目标不同，间隔和超时设计完全不同。保活可以 15–30s；状态同步通常 1–5s；异常检测需要结合任务 SLA，而不是固定间隔。

**2. 根据通道选择轮询还是推送**

- 轮询：适合低频状态同步、后台巡检、宿主只提供 HTTP API 的情况。优先使用 `If-Modified-Since` / `ETag`，只拉变化。
- 推送：适合事件驱动、亚秒级响应、MCP streamable HTTP 或 SSE 可用的情况。优先 SSE（单向、实现简单）；只有确实需要双向交互时才上 WebSocket。
- 混合：推送为主，轮询兜底。推送断线时切换到低频轮询，恢复后重新订阅。

**3. 做应用层心跳，不要只依赖 TCP keepalive**

推送通道必须有自己的 ping/pong 或心跳帧。通常 15–30s 一次，连续 2–3 次未响应就主动重连。SSE 场景可以定时发送注释行 `: ping` 保持连接；WebSocket 可以用标准 ping/pong 或业务心跳。

**4. 事件要带序号和时间戳**

推送事件可能乱序、重复、丢失。每个事件至少包含 `seq`、`ts`、`type`、`payload`。消费端按 `seq` 去重，发现缺口时触发一次全量拉取，而不是继续猜测。

**5. 轮询要加退避和 jitter**

固定间隔轮询在服务端抖动或客户端并发高时会放大问题。可以设置基础间隔 2s，按空转次数指数退避到 30s；多个客户端加随机 jitter，避免同时打到同一个接口。

## 踩坑点

- **代理和网关超时**：Nginx、负载均衡、云函数网关经常 60s 左右断开空闲连接。推送连接必须在 30s 内发一次心跳，并配置 `proxy_buffering off`、`X-Accel-Buffering: no`。
- **SSE 被缓冲**：部分 HTTP 客户端会缓冲响应，服务端需要先发送响应头并 flush，否则客户端迟迟收不到事件。
- **WebSocket 鉴权麻烦**：浏览器 WebSocket 不能自定义 header。不要把 token 长时间放在 URL query 里，优先用首条消息鉴权，或者用子协议传 token。
- **轮询扫全表/全任务**：每次轮询都扫描全量任务队列，会导致数据库和 MCP server 压力线性增长。用 `updated_at > cursor` 或游标分页。
- **把等待外部事件写进阻塞循环**：Agent 主循环里如果 `while (!done) sleep(2s)`，会占用上下文、难以取消。应把等待封装成工具调用的 `pending` 状态，让运行时调度，而不是阻塞。
- **推送事件重复**：重连后如果不带 `Last-Event-ID` 或游标，会重复消费。消费逻辑必须是幂等的。

## 可复用建议

- 封装一个 `HeartbeatAdapter`，把 polling / SSE / WebSocket 的实现差异藏起来。接口上只暴露 `start`、`stop`、`onEvent`、`onTimeout`。
- 事件结构统一，方便回放和诊断。至少要有 `seq`、`ts`、`type`、`payload`。
- 推送优先 SSE，只有需要双向交互时才用 WebSocket；如果已经接入 MCP streamable HTTP，优先复用这个通道。
- 对心跳做指标采集：RTT、丢包、重连次数、轮询空转率。这些指标比“感觉实时”更有用。
- 给所有长任务设置兜底轮询：即使推送可用，也保留一个低频 reconcile 任务，用最终一致弥补事件丢失。

## 总结

heartbeat 不是简单的“定时请求”，它是 AI 助手与外部系统之间的分布式状态同步。轮询胜在简单可靠，推送胜在实时低延迟。工程上更稳妥的做法是：明确延迟目标后选择主通道，用应用层心跳和序号保证可靠，再用低频轮询兜底。对 OpenClaw/Agent/MCP 这类既要实时性又要稳定性的场景，混合方案通常是更务实的选择。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c8528f7ef4c96143.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/4200118a4c7cb599.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/46048e46309f5e7c.png)

