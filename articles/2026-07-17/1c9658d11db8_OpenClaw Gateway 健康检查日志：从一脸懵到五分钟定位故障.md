---
title: OpenClaw Gateway 健康检查日志：从一脸懵到五分钟定位故障
feedId: 29441
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

在 OpenClaw 体系中，Gateway 负责将 MCP 请求路由到后端 agent 或插件服务。为了让调用方不把流量打到已挂掉的节点，Gateway 会持续对所有上游（upstream）执行健康检查，并把结果以结构化日志的形式输出。这些日志看似平淡，却是定位“服务明明活着却返回 502”“间歇性超时”等问题时最直接的线索。

但实际工作中发现，很多同学并不清楚这些日志的语义，看到 `health_check failed` 就认为服务挂了，或者忽略了 `status=200` 但延迟异常的信号，导致误判和过长的 MTTR。这篇文章从工程化视角，梳理一套可复用的健康检查日志阅读与分析思路。

## 问题

你很可能在排障时见过类似这样的日志行：

```
{"timestamp":"2025-01-21T10:23:45.123Z","level":"info","component":"health_checker","upstream":"agent-2:8080","endpoint":"/health","status":200,"latency_ms":1432,"error":"","message":"health check passed"}
```

或者更让人紧张的：

```
{"timestamp":"...","level":"warn","component":"health_checker","upstream":"plugin-x:9090","endpoint":"/ready","status":0,"latency_ms":5001,"error":"context deadline exceeded","message":"health check failed"}
```

第一反应往往是“健康检查失败了，赶紧重启服务”。但在 OpenClaw Gateway 的配置下，一次失败可能只是触发了重试窗口的开端，真正的上下线行为受 `interval`、`timeout`、`unhealthy_threshold`、`healthy_threshold` 等多个参数共同控制。不读懂日志细节，就容易陷入“见红就慌”的陷阱。

## 做法/步骤

### 1. 确认日志来源与格式

OpenClaw Gateway 默认将健康检查日志输出到 stdout，并由日志收集器（如 vector/fluent-bit）统一收集。关键字段通常是固定的：

- `upstream`：被检查的上游标识（IP:端口或服务名）
- `endpoint`：健康检查路径，如 `/health`、`/ready`
- `status`：HTTP 状态码，TCP 检查则为 0（失败）或 -1（超时）
- `latency_ms`：本次检查耗时（毫秒）
- `error`：错误描述（连接拒绝、超时、TLS 握手失败等）
- `message`：本次检查结果摘要

建议在日志平台先按 `component=health_checker` 过滤出健康检查相关条目，再按 `upstream` 分面统计，快速看出问题集中在哪些后端。

### 2. 理解“失败”不等于“下线”

一次 `message: "health check failed"` 只代表一次探测失败。Gateway 内部会维护一个失败计数器，只有连续失败次数达到 `unhealthy_threshold`（通常配置为 3 或 5）之后，才会将该 upstream 标记为不健康，并从负载均衡池中摘除。同样，恢复时需要连续成功 `healthy_threshold` 次才会重新加回。

因此当看到一条失败日志时，先别急着操作，而是回溯最近 3～5 次同 upstream 的日志。如果失败是偶发的（如每隔十几分钟一次，其余都成功），很可能是网络抖动，节点本身并未下线；如果日志里连续出现 `status=0` 或 `connection refused`，那才意味着节点真的需要介入。

### 3. 关注 `latency_ms` 和 `status` 的组合

很多健康检查端点仅仅返回 200，但背后的中间件或数据库可能已经变慢。如果 `status=200` 但 `latency_ms` 持续增大（如从平均 20ms 涨到 1000ms），日志不会报错，但实际调用已经处于降级边缘。我在生产环境遇到过 agent 服务内嵌了一个大模型调用，/ready 虽返回 200，但内部队列积压导致健康检查耗时 3 秒以上。此时 Gateway 仍将其视为健康，直到外部调用超时率飙升。

建议对 `latency_ms` 设置接近业务 timeout 1/3 的告警阈值。比如你的 agent 请求超时为 3 秒，那么健康检查延迟如果连续超过 1 秒，就应该触发一条告警，提醒团队关注后端饱和度。

### 4. 借助日志模式定位典型故障

整理一个简单的决策表，可以快速归类问题：

- `error: "dial tcp ... connection refused"` → 进程未监听端口，可能挂了或被防火墙拦截
- `error: "context deadline exceeded"` 且 `latency_ms ≈ timeout` → 服务响应过慢或端点阻塞，高概率为应用层瓶颈
- `error: "tls: handshake timeout"` → 证书或 mTLS 配置异常
- `status=503` → 服务显式返回不可用，可能是主动下线或依赖不健康
- `status=200` 但 `latency_ms` 极高 → 假健康，需深入检查服务内部指标

把这些模式做成日志平台的快速筛选标签，可以大幅缩短排障时间。

## 踩坑点

1. **健康检查路径选择不当**：用 `/health` 只返回 web 框架存活，不检查数据库或消息队列。一旦依赖组件挂了，业务已受损但 Gateway 仍视为健康。建议对核心 agent 使用 `/ready`，内部串联关键依赖的快速探活。
2. **timeout 与 interval 的比例失调**：如果 timeout 设为 5 秒，interval 也只有 5 秒，一旦服务变慢，健康检查请求会堆积，Gateway 可能瞬间把所有 node 标记为不健康，造成雪崩。一般推荐 interval 是 timeout 的 2～3 倍。
3. **日志级别误设为 info**：生产环境建议将健康检查失败日志设为 warn，成功日志设为 debug。若全部用 info，失败信息容易被淹没。调整 Gateway 的 logger 配置即可。
4. **忽略冷启动窗口**：新启动的 agent 可能需要数秒才能接受请求。如果 unhealthy_threshold 太小，节点会被过早摘除。可配合 `initial_jitter` 或适当调大失败阈值。

## 可复用建议

- **标准化日志字段**：确保所有 team 的健康检查日志都包含 `upstream`、`status`、`latency_ms`、`error`。跨服务排障时才能一致分析。
- **建立健康检查看板**：基于日志聚合系统（如 Grafana Loki + Promtail），构建一个 upstream 维度的面板，显示成功率、P99 延迟、失败错误分布。配合 alert rule，实现“spike in health check failures”的告警。
- **自动化测试健康检查语义**：在 CI 中加入集成测试，验证 `/ready` 在依赖故障时能否正确返回非 200。避免“上线以为有健康检查，实际什么都没覆盖”。
- **日志采样与保留**：健康检查日志量大，设置合理的保留期（如 7 天）和成功日志采样率，降低存储成本的同时保留事故现场的完整失败日志。

## 总结

OpenClaw Gateway 的健康检查日志是系统韧性的“脉搏”。读得准，你能在用户感知到报错前就发现后端变慢或依赖故障；读得偏，就容易误判节点状态、延长恢复时间。核心心法是：**不要只看一句失败，要结合阈值、连续性和延迟趋势做判断；不要把 200 等同于健康，要把延迟纳入监控范畴；不要孤立看 Gateway 日志，要和 agent 内部指标互为印证。** 照着这个思路调整你的日志收集和告警规则，下次再遇到上游抖动时，你就能在五分钟内做出准确决策，而不是盲目重启。

---

