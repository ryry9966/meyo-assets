---
title: OpenClaw Gateway 健康检查日志解读指南
feedId: 29271
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

在基于 OpenClaw 搭建的 Agent 网关体系中，Gateway 承担着所有外部请求的入口与路由职责。为保证后端 Agent 节点（MCP 插件、自动化 Worker 等）的可用性，Gateway 内置了完善的多层次健康检查机制，包括定时主动探测和基于真实流量的被动探测。一旦节点被标记为不可用，Gateway 会立即将其从活跃池中剔除，避免请求打到故障实例。

但在日常运维中，健康检查相关的日志往往输出频繁且信息密集，不少实践者在排查节点异常时容易忽略关键字段，甚至误判“短暂降级”为“永久不可用”，引发不必要的重启或扩容。本文将基于真实工程环境，梳理如何看懂 OpenClaw Gateway 的健康检查日志，并给出可复用的处理模式。

## 问题：日志里的“unhealthy”并不总是故障

典型困惑包括：
- 日志中出现 `unhealthy` 后，节点并未真正下线，或很快恢复，不知道以哪条为准。
- 被动检查触发的降级与主动健康检查失败的错误信息混在一起，难以定位根因。
- 日志量大，全量 `grep` 会淹没在正常 `healthy` 行中，缺少高效的过滤方式。

这些问题几乎都源于对健康检查日志模型的理解不够透彻。下面通过示例拆解看板。

## 做法与步骤

### 1. 定位与确认日志输出配置

OpenClaw Gateway 的健康检查日志默认输出到 `gateway.log`，路径通常在 `$OPENCLAW_HOME/logs/gateway.log` 或容器标准输出。确认日志级别中 `health-check` 相关的 logger 是否为 `INFO` 或更低，否则可能丢失被动检查的详细记录。推荐在 `gateway.yaml` 中显式配置：

```yaml
logging:
  level:
    com.openclaw.gateway.health: INFO
  output: json   # 结构化日志
```

结构化日志（JSON）会在后期自动化处理时方便很多，每条健康检查事件都可被直接解析。

### 2. 理解单条日志的关键字段

一条典型的 JSON 健康检查日志类似：

```json
{
  "ts": "2025-01-12T10:23:45.123Z",
  "logger": "health-check",
  "level": "INFO",
  "node": "agent-worker-3",
  "status": "healthy",
  "latency_ms": 12,
  "type": "active",
  "endpoint": "http://agent-worker-3:8080/health",
  "consecutive_failures": 0,
  "reason": null
}
```

字段含义：
- `status`：**healthy**（正常）/ **degraded**（降级，通常由被动检查触发）/ **unhealthy**（不可达或超过失败阈值）。
- `latency_ms`：响应耗时。超出 `gateway.health.timeout` 则会记为一次失败。
- `type`：`active` 为定时主动探测；`passive` 为真实请求转发过程中发现失败后插入的检查结果。
- `consecutive_failures`：连续失败计数。这是判断节点状态切换的核心指标，达到 `gateway.health.max-failures`（如 5）后状态会从 healthy 变为 unhealthy。
- `reason`：失败原因，如 `timeout`、`connection refused`、`5xx` 等。

### 3. 快速过滤与状态变化追踪

日常排查时，不建议直接 `grep "unhealthy"`，因为该状态可能由被动检查产生，节点在下一秒即恢复。更有效的方法是关注**状态转换边界**和**连续失败计数变化**。

例如使用 `jq` 解析结构化日志：

```bash
cat gateway.log | jq 'select(.logger=="health-check" and .status!="healthy")' \
  | jq -r '[.ts, .node, .status, .consecutive_failures, .reason] | @tsv'
```

可以得到类似下表：

```
2025-01-12T10:23:47.567Z  agent-worker-5  unhealthy  5  timeout
2025-01-12T10:24:17.932Z  agent-worker-5  healthy    0  null
```

如果 `unhealthy` 行在数秒后(或下一次主动探测周期)就出现对应的 `healthy`，说明是短暂抖动，不需人工介入。只有当 `unhealthy` 持续多个周期，且 `consecutive_failures` 未清零时，才应触发告警。

### 4. 区分主动探测与被动探测的作用域

主动探测的间隔固定（如 30s），日志中会出现大量周期性 `healthy` 行，用于整体健康趋势监控。被动探测则较稀疏，但在高峰期能更早暴露节点隐性降级——例如某个节点因 GC 停顿导致请求超时，但主动探测尚未执行。此时会出现一条 `status: "degraded"`、`type: "passive"` 的日志。**degraded 仅由被动检查生成**，表示单次真实流量失败但未达剔除阈值，这属于正常保护机制，不应直接干预。

## 踩坑点

- **误将 passive degraded 当作主动探测失败**：两者触发机制不同，处理策略不同。被动降级需结合业务请求耗时判断，可能是下游瞬时压力所致；主动探测持续失败则更可能是节点进程异常。
- **日志采样导致关键缺失**：部分生产环境为减少 IO 会对健康检查日志采样。一旦采样丢弃了连续的 `unhealthy` 记录，就会造成“节点已被摘除但日志无反映”的假象。务必保证对 `status` 非 healthy 的事件全量记录。
- **忽略恢复日志**：只看到错误，未检查同一节点是否有 `healthy` 恢复。判断节点是否曾真实不可用的最佳证据是“consecutive_failures 达到 max-failures 的 unhealthy 事件”以及“后续没有在短时间内恢复”。
- **将连接拒绝与超时等同处理**：`reason=connection refused` 通常意味进程未监听端口，可以重启；`reason=timeout` 则可能是负载高或死锁，需要压测或线程 dump，而非直接重启。

## 可复用建议

1. **强制结构化输出**：Gateway 日志使用 JSON 格式，并在收集端（如 vector、fluentbit）直接提取 `status`、`node`、`consecutive_failures` 作为监控指标标签。
2. **告警策略设计**：对同一节点，在 5 分钟内出现连续 3 次 `status=unhealthy` 且未有中间 `healthy` 插入时，再触发告警。避免因被动检查瞬时失败产生噪音。
3. **保留原始日志至少 7 天**：健康检查日志是事后复盘节点故障的关键证据，配合请求日志的 `trace_id` 可还原“先有被动失败还是先有主动失败”的完整时序。
4. **编写日常巡检脚本**：例如每小时向运维频道推送一次异常摘要，只展示当前处于 unhealthy 或 degraded 且未恢复的节点，而非全量日志。

## 总结

OpenClaw Gateway 的健康检查日志本质上是一份“节点可用性状态机”的详细记录。读懂它并不需要复杂的工具，关键是要理解 `status` 的状态切换条件、`consecutive_failures` 的累积逻辑以及主动/被动检查的差异。按结构化方式消费这些日志，建立分层过滤与告警规则，就能在噪音中准确捕获真正需要人工介入的异常，显著减少因误判导致的无序操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/f70825e40700fd23.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/785fabcda5e1e6e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/3204d5e4a773c5ad.png)

