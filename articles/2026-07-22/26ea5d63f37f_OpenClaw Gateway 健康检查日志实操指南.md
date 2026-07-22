---
title: OpenClaw Gateway 健康检查日志实操指南
feedId: 30072
source: 综合讨论
publishedAt: 2026-07-22
---

## 为什么需要关注健康检查日志

在 OpenClaw 的分布式 agent 与 MCP 插件体系中，Gateway 是统一的流量入口，负责路由、认证、限流和负载均衡。为了保证请求不被转发到已失效的后端，Gateway 会持续对上游服务（agent runtime、tool server、MCP endpoint 等）执行健康检查。这些检查的结果直接写入日志，是排查服务雪崩、间歇性超时、路由异常的第一手材料。然而，很多团队把健康检查当作“只要不报警就不用看”的组件，直到线上出现 502 或 503 后才回头翻日志，往往已经丢失了最早的异常信号。

## 典型日志长什么样

以结构化 JSON 日志为例，一次成功的主动健康探测通常会输出类似字段：

```json
{
  "ts": "2025-04-10T09:12:03.421Z",
  "level": "debug",
  "msg": "health check passed",
  "service": "mcp-tools-vision",
  "endpoint": "http://mcp-vision:8080/health",
  "status": 200,
  "latency_ms": 3.2,
  "up": true
}
```

如果后端异常，会看到：

```json
{
  "ts": "2025-04-10T09:13:15.287Z",
  "level": "warn",
  "msg": "health check failed",
  "service": "agent-worker-nl2sql",
  "endpoint": "http://nl2sql:8080/health",
  "error": "context deadline exceeded",
  "latency_ms": 5001,
  "up": false
}
```

很多实现还会在连续失败达到阈值后触发熔断，此时日志中常出现 `circuit breaker opened`，说明 Gateway 已主动摘除该节点。

## 如何高效阅读与过滤

### 1. 定位健康检查相关行

在日志平台或本地文件中，推荐使用以下过滤策略：

- 按关键词 `health` 或 `probe` 过滤，避免误命中业务日志中的 `health` 字段，可组合 `msg:"health check"`。
- 若日志量过大，只关注 `level:warn` 或 `error`，但初次排查时建议保留 `debug` 以观察探测频率和延迟趋势。
- 按服务名分组：`service:"agent-worker-nl2sql"` 可以快速锁定具体后端的健康状态变化。

### 2. 从一次失败中提取关键信息

拿上文的失败示例来说，`error:"context deadline exceeded"` 和 `latency_ms:5001` 几乎可以断定探测超时（假设默认超时 5 秒）。此时应该：

- 检查被探测服务是否存活（进程、容器、端口）。
- 是否存在网络策略阻断（如 Kubernetes NetworkPolicy 或安全组）。
- 后端服务的 `/health` 端点是否开启了耗时依赖检查，如数据库连接，导致探活超时。
- Gateway 侧的探测超时是否小于后端实际响应时间，可调整 `timeout` 配置。

### 3. 观察时序规律

使用日志可视化或命令行统计失败频率：

```bash
cat gateway.log | grep "health check failed" | awk '{print $1}' | uniq -c
```

若每隔 N 秒固定出现一次失败，往往是探测间隔与后端重启周期重合。若失败集中在业务高峰期，则可能是资源瓶颈导致健康接口响应变慢，应考虑独立探活端点（如 `/health/readiness` 仅检查自身依赖）。

## 踩坑点与经验

### 陷阱一：日志级别配置不当

开启 `debug` 级别时，每秒数十次探测会产生海量日志，淹没真正的错误。建议生产环境将健康检查日志保持 `info` 或 `warn`，仅记录失败和状态变化（up 从 true 变为 false 时打印）。部分网关支持采样策略，比如每 10 次成功输出一条 summary，减少噪音。

### 陷阱二：主动探测 vs 被动探测混淆

OpenClaw Gateway 可能同时使用主动健康检查（定期发送探活请求）和被动健康检查（根据真实请求的响应判断）。后者通常不会产生独立日志，但如果开启了“outlier detection”（异常点检测），需确认背后是否依赖主动探测，否则连续错误可能无日志记录，导致排查盲区。阅读日志时先弄清楚是哪种机制产生的记录。

### 陷阱三：熔断后静默

一旦 circuit breaker 打开，Gateway 可能停止发送探活请求，日志里不再出现该服务的健康检查记录。这种静默很容易让人以为服务已恢复。配置中最好开启 “half-open” 状态的半探活，并确保半开时的探测也写入日志，同时为 circuit breaker 事件单独设置通知。

## 可复用的工程化建议

1. **结构化日志 + 统一采集**：确保健康检查日志为 JSON 格式，接入 Loki、Elasticsearch 等，建立按服务、错误类型、延迟分位数的面板。
2. **告警前置**：不要等用户报障。基于日志设置告警规则：连续 3 次探测失败或探测延迟 > p99 阈值即通知。
3. **探活端点设计**：后端提供轻量级的 `/health` 接口，避免依赖外部资源；若需深度检查，另开 `/health/deep`，避免影响网关的路由决策速度。
4. **关联 trace**：在探活请求中注入自定义 header（如 `x-health-check: true`），并在 agent 端识别后跳过分录 span，或单独生成链路，方便在分布式追踪中过滤。
5. **保留历史快照**：将健康检查状态变化写入时序数据库（如 Prometheus 指标），配合日志回溯，比单独 grep 更直观。

## 总结

OpenClaw Gateway 的健康检查日志不是可有可无的调试信息，而是系统韧性的实时观测窗口。通过学会定位、解析和关联这些日志，你可以在异常扩散到业务之前，快速定位是某个 MCP 工具未就绪、agent worker 内存溢出还是网络分区。与其在告警风暴中盲查，不如从第一个 `health check failed` 开始，把它当作排障的发令枪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/d9554b456f4e77fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/50ae2a1a38d77e4d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/dc4fc8909041cc55.png)

