---
title: OpenClaw Gateway 健康检查日志：从看懂到可排障
feedId: 29890
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在 OpenClaw 的部署拓扑里，Gateway 是流量入口，也是外部负载均衡、Kubernetes readiness/liveness 探针的直接接触点。一旦 Gateway 的健康检查日志不清晰，排查时就会被“服务正常但负载均衡一直摘流”这类问题拖住。

很多团队上线后只看 `/health` 的 HTTP 状态码，很少去读 Gateway 自身的健康检查日志。实际上，Gateway 往往不只是做 200/非200 的简单返回，它可能会对上游的 Agent、MCP 端点、插件模块做级联探测。这些探测的细节、超时、部分失败，都藏在日志里。

## 健康检查日志常见的三类问题

1. **“200 假阳性”**  
   Gateway 返回 200，但日志里显示对某个 Agent 的 gRPC health check 超时，或者插件管理器处于降级状态。如果只看状态码，会以为一切正常。

2. **“日志淹没”**  
   默认每分钟一次的健康检查，在全量 DEBUG 或 INFO 级别下产生大量重复日志，把真实错误冲走。有人配了日志采样，但把错误采样掉了。

3. **“静默降级”**  
   Gateway 自身进程存活，但内部某些模块（如 MCP 连接池）在健康检查逻辑里被标记为 WARN，却不影响整体返回码。如果没有针对日志级别的监控，这类降级可以持续数天不被发现。

## 做法/步骤

### 1. 定位健康检查日志来源

OpenClaw Gateway 的健康检查日志不是单一文件。通常有三个来源：

- **HTTP 接入层日志**：来自反向代理或内置 HTTP server，记录 `/health` 请求的方法、状态码、耗时。
- **健康探针执行日志**：由内置 health checker 组件输出，包含对各个依赖（Agent、插件、MCP 服务）的探测结果。
- **模块自检日志**：插件注册中心、连接池、配置重载等模块在健康检查周期内主动写入的 WARN/ERROR。

先确认日志配置中 `health` 或 `probe` 相关的 logger name，例如 `openclaw.gateway.health`。将其单独输出为一个文件或 stdout stream，便于 grep 和分析。

### 2. 理解一条典型日志的结构

一条结构化的健康检查日志通常包含以下字段（假设 JSON 格式）：

```json
{
  "ts": "2025-03-28T10:00:01.234Z",
  "level": "info",
  "logger": "openclaw.gateway.health",
  "msg": "health check completed",
  "overall": "healthy",
  "components": {
    "agent-upstream": "ok",
    "mcp-connector": "degraded",
    "plugin-registry": "ok"
  },
  "duration_ms": 230,
  "probe_id": "a1b2c3d4"
}
```

关键点：
- `overall` 是 Gateway 最终返回给负载均衡的状态依据，通常由 `components` 的严重等级决定。
- `components` 中的状态不止 `ok` 和 `error`，会有 `degraded`、`unknown`、`timeout`。`degraded` 不代表整体不健康，但需要关注。
- `duration_ms` 连续升高可能预示着上游 Agent 或 MCP 服务响应变慢，即便状态仍为 `ok`。
- `probe_id` 可以用作全链路关联，当一次 `/health` 请求触发级联探测时，所有子探测共享同一个 probe_id。

### 3. 设置合理的日志级别与采样

不要全局开启 DEBUG 来记录健康检查。建议：

- 健康检查成功记录为 INFO，且采样记录，例如每 5 分钟打印一条摘要性日志（包含各组件状态统计），而非每次探测一条。
- 健康检查失败或首次变为 `degraded` 时，以 WARN 输出完整 component 详情，且不采样。
- `timeout` 类错误应带有超时阈值和实际耗时，方便排查网络或上游限流。

示例日志策略：
```
logger.openclaw.gateway.health = INFO
sampling.health.success = 1/10
```

### 4. 关联指标与日志

单靠日志无法发现趋势。将 `overall` 状态与 `components` 中各项的降级次数输出为指标（如 Prometheus counter/gauge），然后配合日志排查具体原因。

典型指标：
- `openclaw_gateway_health_overall_status`：0/1 表示不健康/健康。
- `openclaw_gateway_health_component_status{component="mcp-connector"}`：0=ok, 1=degraded, 2=error。
- `openclaw_gateway_health_probe_duration_ms`：直方图，观察 P95/P99。

当指标显示 `mcp-connector` 频繁变为 degraded 时，回到日志里搜索 `"mcp-connector" AND ("degraded" OR "timeout")`，结合 probe_id 下钻。

## 踩坑点

### 坑1：健康检查缓存导致日志滞后

一些实现为了降低探测开销，会将健康检查结果缓存数秒。如果在这段时间内组件状态变化，日志可能不反映最新情况。排查问题时要关注日志中的 `cached: true` 标记，必要时直接调用强制刷新端点（如 `/health?force=true`）。

### 坑2：认为没有 error 日志就是健康

很多降级只打 WARN，不触发 error。要建立对关键组件的 `degraded` 状态的告警，而不是只盯着 `overall` 为 `unhealthy`。

### 坑3：日志时间戳与负载均衡探测周期错位

负载均衡器探测间隔通常是 5~30 秒，Gateway 的日志可能每分钟才打一次摘要。中间如果有瞬时故障，日志里可能看不到。因此需要在负载均衡侧也记录探测失败日志，两边时间对齐排查。

## 可复用建议

- **结构化日志是底线**：确保健康检查日志输出为 JSON，字段规范，尤其是 `components` 用嵌套对象而不是字符串拼接。
- **分离健康日志流**：把健康检查日志路由到独立的低保留期的日志流，避免占有主日志存储。
- **主动拉取而非被动等待**：在排障时，用 `curl` 连续请求 `/health` 并同时 `tail -f` 健康日志，观察 probe_id 关联关系，这比事后翻日志快得多。
- **探针降噪**：对于非关键插件，可配置为“非阻断”探针，其失败仅记录为 WARN 且不影响 overall 状态，但要打日志摘要。

## 总结

OpenClaw Gateway 的健康检查日志不是简单的 200/非200 记录，而是一张描述了所有上游依赖瞬时状态的快照。把日志当成可查询的“健康检查结果数据库”，配合结构化字段、采样策略与指标，才能在降级刚开始时就发现根因，而不是等用户报障再回翻一堆 INFO。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/3effa62da50b66a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/cadf4bca1680d377.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/1e144b0279466f6c.png)

