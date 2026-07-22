---
title: OpenClaw Gateway 健康检查日志解读与排障实践
feedId: 30041
source: 综合讨论
publishedAt: 2026-07-22
---

# OpenClaw Gateway 健康检查日志解读与排障实践

## 背景：为什么关注健康检查日志

在基于 OpenClaw Gateway 的 Agent 与 MCP 插件架构里，健康检查是自动化运维的第一道防线。无论是 Kubernetes 的 liveness/readiness 探针，还是上游负载均衡器的心跳检测，最终都表现为大量对 `/healthz` 或自定义健康端点的请求。OpenClaw Gateway 自身的 runner 会记录每一次健康检查的结果，这些日志不仅是“通/不通”的二元信号，更包含延迟抖动、后端过载、配置漂移等关键信息。

我们会遇到这样的场景：日志文件被健康检查请求刷屏，真正有问题的业务报错被淹没；或者在无告警的情况下，服务节点被容器编排系统重启，事后只能靠网关日志复盘。所以，弄清楚**怎么看、看什么、什么时候需要关心**健康检查日志，是维持服务稳定性的基本功。

## 典型问题：被“正常”日志掩盖的信号

最常见的问题是：健康检查返回 200 且耗时低于 5ms，开发者便习惯性忽略。但以下两类异常往往藏在看似正常的记录里：

1. **间歇性超时**：健康检查平均延迟正常，但 P99 偶尔触及探针超时阈值（例如 1 秒），导致 kubelet 连续探测失败，触发容器重启。这种抖动在聚合指标里不明显，必须查看原始日志的时间序列。
2. **后端健康但网关转发异常**：健康检查直接连接 Gateway 进程的本地端口，始终返回 200；但网关到上游 Agent 的连接池耗尽，实际业务请求已经大量 502。如果健康检查只测“网关进程存活”而不做依赖检查，日志就会给出虚假的“一切正常”。

## 步骤：如何高效阅读健康检查日志

### 1. 确保日志结构化与关键字段

在 OpenClaw Gateway 配置中，将健康检查日志输出为 JSON 并至少包含以下字段：

```json
{
  "timestamp": "2025-01-15T10:23:45.123Z",
  "type": "health_check",
  "route": "/healthz",
  "status": 200,
  "latency_ms": 3.2,
  "upstream": "agent-service:9090",
  "upstream_status": 200,
  "error": ""
}
```

有了结构化日志，就可以用 `jq` 或日志平台快速聚合：

```bash
# 统计 1 分钟内健康检查非 200 的次数
cat gateway.log | jq 'select(.type=="health_check" and .status!=200)' | wc -l
```

### 2. 区分探针类型与日志级别

将 Kubernetes **readinessProbe** 与 **livenessProbe** 设置为不同的路径，并在日志中记录 `probe` 字段。比如：

- liveness：`GET /healthz?probe=liveness`
- readiness：`GET /healthz?probe=readiness`

这样当 Pod 频繁重启时，你能很快过滤出 liveness 检查的失败日志，而不是混在 readiness 因后端启动慢而短时失败的噪音里。同时，建议将健康检查的 2xx 日志级别设为 `debug`，只在失败时提升至 `warn`，防止日志洪流。

### 3. 通过延迟分布定位上游瓶颈

健康检查如果仅探活网关自身，延迟通常稳定在 1ms 以内。如果配置为“透传检查”（网关检查 → 上游 Agent → 依赖数据库），那么延迟的升高就是上游性能劣化的先兆。观察 `latency_ms` 的 P95 趋势：

```
# 假设使用 Loki + LogQL
quantile_over_time(0.95, {type="health_check"} | unwrap latency_ms [5m])
```

当 P95 从 10ms 陡增到 200ms，即使状态仍为 200，也应当排查上游连接池或慢查询。多数重启事故在发生前 3~5 分钟，健康检查延迟就已经开始爬升。

### 4. 关联业务请求日志验证

健康检查成功不代表转发链完整。需将有 `upstream_status` 字段的健康检查日志和实际业务请求日志做关联。例如发现健康检查 `upstream_status` 为 200，但同时间戳附近业务请求 `upstream_status` 大量 502。这种情况通常是健康检查使用了短连接而业务请求使用了长连接，长连接池中的陈旧连接未及时刷新。解决办法：让健康检查也复用与业务相同的连接池，或在健康检查端点中加入一次真实的依赖探测。

## 踩坑点

- **盲目开启全量健康检查日志**：未区分日志级别的生产环境，半小时内磁盘被填满。务必在 Gateway 中限制健康检查的日志级别或启用采样。
- **健康检查路径暴露内部拓扑**：将 `/healthz` 设计为返回上游 IP、端口等信息的调试端点，容易被攻击者利用。生产环境应只返回状态码和简短的 `ok`。
- **将健康检查延迟直接作为业务延迟**：健康检查通常是最轻量的请求，不应直接等价于业务接口性能。需要另外建立 SLO 监控。
- **忽略探针超时与网关超时的衔接**：kubernetes `timeoutSeconds` 设定为 1 秒，但 Gateway 内部超时设置为 5 秒，那么即使上游卡死，健康检查也不会在 1 秒内返回失败。两个超时必须协调：Gateway 应给予健康检查更短的内部超时（如 800ms），预留网络往返。

## 可复用建议

1. **统一日志契约**：在团队内约定健康检查日志必须输出 `type=health_check`、`probe`、`upstream_status` 等字段，形成可查询的观测惯例。
2. **利用采样的全量存储**：在 Gateway 层对 2xx 健康检查日志按 1% 采样保留全量字段，其余 99% 只写一条计数器，兼顾存储成本与排障细节。
3. **构建健康检查的衍生指标**：将原始日志转化为 `health_check_failure_rate`、`health_check_latency_p99` 等 Prometheus 指标，配合 ALERTS 规则：“当 readiness 检查总失败率 >1% 且持续 2 分钟”，而非仅靠重启事后追查。
4. **定期做故障演练**：故意停止一个上游 Agent，观察健康检查日志中是否立刻出现 `upstream_status: 502`，以及告警是否在 30 秒内触发。故障演练能验证整个观测链路的有效性。

## 总结

OpenClaw Gateway 的健康检查日志不应被当作需要“关掉省资源”的噪声，而是低成本的系统健康信号源。工程上要做的，是用结构化日志、合理分级、延迟分布分析和上下游关联，把被动翻日志变为主动发现风险。当你能在日志中看出“10 分钟后这个 Pod 可能会被重启”时，才算真正用好了健康检查这一基础能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/6fcba3edf5f62b5c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/fe91f40ecada56c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/69926da98c9b06c1.png)

