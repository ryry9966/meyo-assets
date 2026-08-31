---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 35608
source: 综合讨论
publishedAt: 2026-09-01
---

# 背景

在 OpenClaw/Agent/MCP/插件/自动化场景里，主控与外部 worker 通常是多进程或多服务结构：MCP server 跑长任务、插件宿主执行脚本、队列 worker 消费任务。主控需要知道这些 worker 是否还活着、任务是否卡住、要不要重试或降级。这就是 heartbeat 要解决的问题。

heartbeat 有两种基本形态：主控轮询（polling）和 worker 推送（push）。两者没有绝对优劣，关键取决于网络拓扑、延迟预算和实现成本。

# 问题：轮询 vs 推送的取舍

轮询：主控定时拉取 `/health` 或 `/tasks/:id/status`。实现简单，worker 不需要维护长连接，适合 worker 在 NAT 后面或只能被动响应。缺点是实时性受间隔限制，worker 多时请求量大，状态没变化时也产生重复请求。

推送：worker 定时向主控发心跳，或通过 WebSocket/SSE 持续连接。延迟低、主控被动接收，适合大量 worker。但要求 worker 能主动访问主控地址，断线重连容易打爆主控，主控还要管理连接和超时。

# 做法/步骤

1. **定义心跳契约**  
payload 保持最小：`worker_id/lease_id`、`seq`（单调递增）、`ts`（参考时间）、`status`（ACTIVE/DEGRADED/BUSY）、`progress`（可选）。日志、上下文不要塞进心跳。

2. **状态机和超时**  
主控维护 `last_seen` map，状态：`ACTIVE -> STALE -> DEAD`。TTL 设为 2-3 个心跳间隔；STALE 先降级观察，DEAD 后再回收或重试，避免网络抖动误判。

3. **选择传输**  
- 能主动连主控：优先推送，用 WebSocket/SSE，或退化到 HTTP POST，间隔 10-30 秒。  
- 只能被动响应：轮询，间隔 5-15 秒，配合 ETag/If-None-Match 减少无效传输。  
- liveness 用推送或轻量轮询，progress 按需查询，不要把大状态放进心跳。

简单伪代码：

推送端：
```
seq = 0
loop:
  if now - last_sent >= interval:
      send({type:"heartbeat", seq, lease_id, ts: monotonic(), status})
      seq += 1
      last_sent = now
  sleep(1)
```

轮询端：
```
while active:
    resp = get_status(task_id, etag)
    if resp == 304: continue
    update(resp.json())
    if now - last_seen > ttl:
        mark_stale_or_retry()
    sleep(poll_interval)
```

# 踩坑点

- **心跳风暴**：大量 worker 固定间隔同时推送，主控被打满。间隔必须加 jitter（±10-20%），或主控下发错峰间隔。
- **只用心跳判断任务健康**：心跳正常不代表任务在推进。MCP tool 可能卡死但心跳还在发。需要 progress 单调变化检测。
- **重连风暴**：推送断开后立即重连，多 worker 同时重连会压垮主控。重连要指数退避 + jitter，例如 1s/2s/4s/8s，上限 30s。
- **时间戳不可靠**：worker 和主控时钟可能不同步。超时判断用主控本地单调时间，不用墙钟；seq 用于检测重排。
- **心跳跑在业务线程**：插件或 MCP server 主循环阻塞会导致心跳发不出。用独立后台 timer/task，必要时加 watch dog 线程。
- **Stdio MCP 特殊**：stdout 被 JSON-RPC 占用，心跳不能直接混入。单独走 SSE/HTTP 控制通道，或使用协议内 ping/pong，不要阻塞 stdout。

# 可复用建议

- 心跳契约必须有 `lease_id` 和 `seq`；状态变更用专门消息，不要让 heartbeat 承载业务状态。
- liveness 间隔 10-30 秒，TTL 2-3 倍；progress 按需查询。
- 推送优先 WebSocket/SSE，退化为 HTTP POST；轮询优先 ETag。
- 主控集中做 last_seen 超时检测，不要在业务逻辑里分散判断。
- 正常心跳日志用 DEBUG，只记录 STALE/DEAD 和恢复事件。
- 提供手动探活/立即触发心跳的运维接口，方便排障。

# 总结

heartbeat 本质是租约和活跃度信号，不是简单的“ping 一下”。轮询适合网络受限、被动 worker；推送适合低延迟、大量 worker。在 OpenClaw/Agent/MCP/插件实践里，建议 liveness 用推送或轻量轮询，progress 按需拉取，并一定加上 jitter、TTL 和重连退避。先定清状态机和超时策略，再选轮询还是推送，能省掉大量排障时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/f879c630bf3f4449.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8d80a2d9c2d81675.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e6a6291d2d0fe2f8.png)

