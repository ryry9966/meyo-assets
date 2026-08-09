---
title: AI 助手的 Heartbeat 设计：轮询 vs 推送的实战取舍
feedId: 32324
source: 综合讨论
publishedAt: 2026-08-10
---

# AI 助手的 Heartbeat 设计：轮询 vs 推送的实战取舍

在 OpenClaw 这类可扩展 AI 助手框架里，Agent 经常需要驱动长时间运行的任务：调用远端工具、等待 CI 结果、监控子进程状态。心跳（heartbeat）机制几乎是这些场景的标配——既要保证任务状态的及时同步，又不能打爆外部 API。一旦选择不当，轻则浪费 token 和带宽，重则导致任务误判、重试风暴。本文从工程落地的角度，梳理在 OpenClaw 插件与 MCP Server 实践中，轮询与推送两种心跳模式的取舍、实现要点和踩坑记录。

## 背景：Agent 为什么需要心跳

拿一个真实的插件需求举例：你让 OpenClaw 助手去触发一次远程模型训练，然后持续汇报进度直到完成。这期间助手需要知道训练是否还在跑、有没有挂掉、进度到了多少。你没办法让训练脚本主动“回来”找助手，于是心跳就登场了。对 MCP Server 来说也是如此——OpenClaw 需要知道某个 MCP 工具提供者是否仍然在线，否则一旦连接断开，Agent 可能长时间无响应。

## 问题：轮询与推送的本质矛盾

- **轮询 (Polling)**：客户端每隔 N 秒请求一次健康检查或状态接口。实现简单，但 N 选小了容易触发 API 限流（HTTP 429），选大了状态延迟不可控。在 OpenClaw 中，如果每个插件独立轮询，资源消耗会线性增长。
- **推送 (Push)**：通过 WebSocket、SSE 或 Webhook 回调，服务端主动通知状态变更。实时性极高，但需要维护长连接或回调可达性，断线、网络抖动、回调丢失是家常便饭。

工程上不能只看理论优势，得结合 OpenClaw 的 MCP 传输层和插件生命周期做权衡。

## 实战做法与步骤

### 1. 基础轮询心跳实现

在 OpenClaw 自定义插件中，可以使用内置的 `setInterval` 风格调度器（或直接用 `asyncio` 轮询）：

```python
async def poll_health(self, url: str, interval: int = 10):
    while not self._stopped:
        try:
            resp = await self.http_client.get(url, timeout=5)
            if resp.status == 200:
                self.update_status("healthy")
            else:
                self.update_status("unhealthy")
        except Exception:
            self.update_status("unreachable")
        await asyncio.sleep(interval)
```

关键点：状态更新写入 OpenClaw 共享上下文中，让 Agent 决策时能读到最新信息。

### 2. 基于 MCP SSE 的推送心跳

如果你的工具提供者是一个 MCP Server，直接利用 MCP 的 SSE 传输层推送心跳事件是最轻量的方式。服务端不断发送 `ping` 事件，客户端只需监听：

```python
# 在 MCP Client 端 (OpenClaw 插件内)
async for event in sse_stream:
    if event.event == "heartbeat":
        last_heartbeat = time.time()
```

无需额外开启 HTTP 轮询，连接保活由 MCP 底层负责。更妙的是，MCP 的 `ping` 协议操作也可以用来主动探测对端是否存活，弥补 SSE 单向推送的不足。

### 3. 混合策略：降级与升档

实际产品里，我们不会只赌一种方式。在一个外部代码审查插件中，最初用 5 秒轮询 CI API，结果被 GitLab 限制了频率。改成优先注册 Webhook 回调，同时在本地挂一个轻量 HTTP 服务器接收回调，作为主心跳链路；当 Webhook 超过 30 秒未到达，插件自动切换到 20 秒间隔的轮询作为兜底。这就是典型的“推送优先、轮询降级”模式。

## 踩坑点

- **轮询频率的陷阱**：别默认 1 秒。不少 SaaS 服务的限流窗口很小。建议从 15~30 秒起步，并读取 API 返回的 `Retry-After` 或 `X-RateLimit-Reset` 头来动态调整。
- **推送的“假死”问题**：WebSocket 连接可能因中间代理空闲断开而关闭，但两端都以为对方还在。一定要在应用层发送心跳帧（ping/pong），并设置合理的超时（如 30 秒无 pong 则重连）。
- **心跳超时 ≠ 任务失败**：心跳只代表通信链路状况，不表示业务任务挂了。Agent 逻辑必须区分 `heartbeat_timeout` 和 `task_timeout`，避免错误终止还在跑的耗时操作。
- **时钟偏移与时间戳**：在推送模式下，不要用服务端时间戳和本地比较，始终用本地接收时刻减去网络 RTT 估算，或使用单调时钟。
- **回调可达性**：内网开发时 Webhook 地址可能不可达。建议配合 OpenClaw 的隧道服务（如果有）或使用反向 WebSocket 连接避免 NAT 问题。

## 可复用建议

1. **配置化一切心跳参数**：间隔、超时、重试次数、退避倍数都做成可配置项，不同环境调优。
2. **使用指数退避重连**：推送断开后，重连间隔采用 `min(interval * 2, max_interval)`，避免重试风暴。
3. **统一状态模型**：在 OpenClaw 中定义一个标准的心跳状态对象，包含 `status`, `last_seen`, `error_message`, `round_trip_ms`，各插件一致使用。
4. **利用 MCP 的 `ping` 机制**：如果双方都是 MCP 实现，直接用协议级 ping 检测存活，无须重复造轮子。
5. **避免心跳风暴**：多插件场景下，可共用一个连接管理器，统一心跳收发，而不是每个插件各自轮询。
6. **监控与告警**：对心跳异常次数做指数移动平均，当恶化时通过 OpenClaw 通知通道告知管理员，而不是静默降级。

## 总结

轮询和推送并非二选一，而是一组可以随场景切换的策略集。在 OpenClaw 的生态里，充分利用 MCP 传输层的能力，可以实现从简单轮询到实时推送的平滑过渡。工程上只要守住“可观测、可降级、可配置”三条底线，心跳机制就能为 Agent 提供可靠的连接感知，而不是变成新的故障源。

---

