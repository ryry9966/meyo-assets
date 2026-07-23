---
title: OpenClaw Gateway 健康检查日志排查指南：从误解到可控的度量
feedId: 30249
source: 综合讨论
publishedAt: 2026-07-24
---

# OpenClaw Gateway 健康检查日志排查指南：从误解到可控的度量

## 背景：健康检查日志为什么值得关注

在 OpenClaw 生态里，Gateway 是所有 Agent、插件与外部工具链之间的流量入口。它内置的健康检查（Health Check）机制用于探测下游服务的存活状态，这些探测行为会持续产生日志。对大部分实践者而言，这些日志要么被直接忽略，要么因为周期性出现的 `503`、`connection refused` 或 timeout 而引发不必要的告警。理解这些日志的结构、来源与配置方式，是把 Gateway 从“黑盒”变成可观测节点的关键一步。

## 问题：看似异常，实则正常的检查日志

常见场景：用户部署了 MCP 插件或自定义 Agent，Gateway 每隔几秒打印一条类似 `health check failed: dial tcp 127.0.0.1:9090: connect: connection refused` 的日志，但服务本身并没有真的宕机。运维同学本能地开始排查网络，或者直接关掉健康检查，行为粗暴，反而掩盖了真实故障。

更有隐蔽的情况：Gateway 配置了多个上游，某个上游的健康检查失败日志被淹没在大量成功日志中，直到插件超时告警才被发现。这说明我们需要一套结构化的读日志方法，而不是依赖告警风暴。

## 步骤：如何系统地阅读健康检查日志

### 1. 定位日志来源与字段
Gateway 健康检查日志通常由内置的 `healthcheck` 模块输出，日志级别为 `info` 或 `debug`。一条典型的 JSON 格式日志可能长这样：
```json
{"time":"2025-03-12T08:15:00Z","level":"info","msg":"health check result","upstream":"mcp-brave","endpoint":"http://127.0.0.1:8080/health","status":"unhealthy","error":"dial tcp: i/o timeout","duration_ms":1500}
```
关键字段：
- `upstream`：被检查的服务名称，对应 Gateway 配置中的 `upstreams[].name`。
- `status`：`healthy` / `unhealthy`，是二值判定。
- `error`：具体的失败原因（DNS 解析、连接拒绝、超时、非 2xx 响应）。
- `duration_ms`：探测耗时，异常升高的延迟通常是上游过载的早期信号。

### 2. 区分探测类型
Gateway 通常支持两种探测：
- **被动探测（Passive）**：基于实际请求的失败率统计，日志中常标记为 `passive` 或 `circuit_breaker`。
- **主动探测（Active）**：周期性的专用健康检查请求，日志条数多，频率固定。

读取时先按类型过滤。如果被动探测日志频繁出现 `unhealthy`，说明真实用户请求已经受损；而主动探测日志的偶发失败，可能只是网络抖动，需要结合持续时间来判断。

### 3. 关联上游实际状态
不要只看单条日志。使用 `jq` 或 Loki/Grafana 面板按 `upstream` 聚合最近 5 分钟的 `unhealthy` 次数：
```bash
cat gateway.log | jq 'select(.msg=="health check result" and .status=="unhealthy")' | jq -s 'group_by(.upstream) | map({upstream: .[0].upstream, count: length})'
```
若某上游持续返回失败，且被动探测同时出现失败，基本可判定组件宕机。若仅主动探测偶尔失败，且 `duration_ms` 在阈值附近波动，则要检查超时设置和链路重试配置。

## 踩坑点

**坑1：把主动探测失败等同于服务不可用**
主动探测通常使用独立的探活端点（如 `/health`），但该端点可能只是进程级存活标志，并不代表业务能力。出现 “connection refused” 时，可能是上游的探活端口与业务端口分离，或容器刚启动仍在预热。请务必核对 `endpoint` 是否与真实业务地址一致。

**坑2：忽略健康检查的全局配置**
Gateway 的默认探测间隔往往很短（如 5s），且重试阈值很紧（如 2 次失败就标记 unhealthy）。在分布式部署中，稍大的网络延迟就会触发周期性的假阳性日志。调整 `interval`、`timeout` 与 `unhealthy_threshold` 时，要同步修改日志过滤规则，否则 Grafana 图表上全是红点。

**坑3：日志级别导致关键信号丢失**
部分实践者为了抑制噪点，直接关闭健康检查日志，或视而不见。正确的做法是保留 `info` 级别，但通过结构化日志采集工具对 `unhealthy` 日志单独聚合并设置合理的告警窗口（如连续 3 次失败才告警），而不是基于单条日志触发。

## 可复用建议

- **建立 per-upstream 健康看板**：为每个 upstream 创建被动失败率、主动探测失败次数、平均探测延迟三条曲线，设定不同颜色的阈值带。
- **编写日志解析脚本**：一个简单的 Shell 或 Python 脚本，从 Gateway 本地日志中提取近 1 分钟的 unhealthy 摘要，作为部署流水线的 smoke test 步骤。
- **定制探活端点**：如果上游组件无法改造，可在 Gateway 的 `health_checks` 配置里将主动探测的 `path` 改为一个轻量级业务接口（例如 `/api/status`），让健康检查变成真正的“商业逻辑存活”信号。
- **利用 MCP 插件的心跳**：当 Gateway 对接 MCP 服务器时，可以利用 MCP 的 `ping` 机制替代 TCP Connect 探测，获取更精准的协议层健康状态，并输出带 `protocol=mcp` 标签的日志。

## 总结

OpenClaw Gateway 的健康检查日志既不是故障警报器，也不是可有可无的背景噪音。它是系统脉搏的量化表达。读懂它的关键在于：结构化字段过滤、主动/被动探测分离、结合实际请求影响度设置告警阈值。当你不再被偶发的 `unhealthy` 吓到，而是能通过延迟趋势提前发现上游性能拐点时，日志的价值才真正体现出来。最终目标，是让健康检查日志成为可观测性拼图里安静但敏锐的那一块，而不是运维的焦虑来源。

---

