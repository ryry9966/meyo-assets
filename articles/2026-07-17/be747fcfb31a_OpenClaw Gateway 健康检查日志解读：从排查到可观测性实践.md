---
title: OpenClaw Gateway 健康检查日志解读：从排查到可观测性实践
feedId: 29355
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景
在 OpenClaw Agent、MCP 插件或自动化流水线的部署拓扑中，Gateway 承担着路由、鉴权与健康检查的角色。一旦某个插件服务被打上 `unhealthy` 标签，流量就会摘除，甚至触发重启，这会直接中断 Agent 的调用链。运维同学的直觉反应是去看 Gateway 的健康检查日志，但这些日志里的字段含义、失败模式经常被误读，导致排障走了弯路。本文将从一个真实场景出发，系统梳理健康检查日志的阅读与分析方式，并沉淀可复用的可观测性实践。

## 典型问题
某 MCP 插件在滚动更新时，Gateway 日志持续输出：
```
health-check target=mcp-search type=readiness status=unhealthy error="dial tcp 10.0.1.12:8080: connect: connection refused"
```
运维看到 `connection refused` 立刻重启 Pod，但情况没有改善。实际上该插件依赖一个初始化脚本，端口在容器启动后约 15 秒才能监听，而 readiness 检查从第 5 秒就开始执行，频繁被检导致一直摘流，形成“启动越慢越无法上线”的死循环。日志里的 `connection refused` 指向的是端口未就绪，而非进程崩溃。

## 做法/步骤
### 1. 开启详细日志
Gateway 默认健康检查日志可能只有 `unhealthy` 状态，缺少耗时、错误详情。调整配置：
```yaml
health_checks:
  logging:
    level: debug          # 或 trace
```
重启 Gateway 后，每条健康检查都会输出完整条目。

### 2. 看懂一条日志
```
[2025-01-02T08:30:15.123Z] health-check target=agent-executor type=liveness status=unhealthy \
  error="Get \"http://10.0.2.5:9090/healthz\": context deadline exceeded" duration=2.002s
```
关键字段：
- `target`：后端服务名，对应配置中的 upstream。
- `type`：`liveness`（存活探针，失败会重启容器）、`readiness`（就绪探针，失败仅摘流）、`startup`（启动探针）。
- `status`：`healthy` / `unhealthy`。
- `error`：具体失败原因，这是排障的核心。
- `duration`：检查耗时，用于判断超时与性能。

**注意**：`liveness` 失败不等于服务宕机，可能只是因为健康检查路径写错或超时阈值过小；`readiness` 失败时容器仍在运行，不应盲目重启。

### 3. 分析常见失败模式
将 `error` 字段归类，可快速定位方向：

- `connection refused`：端口未监听，常见于启动初期、进程崩溃、安全组/iptables 阻拦。
- `context deadline exceeded`：健康检查超时。检查超时参数与后端实际响应延迟，逐步慢的应用需调大 `timeout` 或使用 `startup` 探针。
- `HTTP 503`：健康检查端点返回不可用，通常意味着依赖组件（数据库、Redis）未就绪。
- `TLS handshake error` / `x509: certificate`：证书问题，检查 mTLS 配置或证书有效期。
- `no such host`：DNS 解析失败，确认 Service 名称或 CoreDNS 状态。

### 4. 聚合日志做趋势判断
单条日志看不出规律，借助 `jq`（若日志为 JSON）或 `awk` 统计：
```bash
grep "health-check" gateway.log | jq -r '[.target, .type, .status] | @tsv' \
  | sort | uniq -c | sort -rn
```
搭配时间窗口过滤，可以判断是偶发抖动还是持续故障。

### 5. 结合指标与端点响应
在排障时，除了日志，还应直接 curl 健康检查端点，确认返回内容；同时观察 Gateway 暴露的 Prometheus 指标（如 `gateway_health_check_failures_total`），看失败计数曲线。日志、指标、端点三重验证，避免“日志里失败但服务实际正常”的误导（例如网络策略只阻断 Gateway 但 Pod 自身健康）。

## 踩坑点
- **混淆探针类型**：对 `readiness` 失败执行重启操作，导致滚动更新雪崩。
- **误判 `connection refused`**：Kubernetes 环境中，Pod 未就绪时的拒绝是常态，需检查 `startup` 探针设置。
- **日志截断**：某些 Gateway 实现会截断长错误信息，需要提高日志输出长度或改用 `stderr` 打印。
- **缺少关联上下文**：没有注入 `trace_id`，当健康检查失败时无法关联到具体请求链路，难以复现偶发性故障。

## 可复用建议
- **轻量健康检查端点**：仅返回 200 和 `{"status":"ok"}`，避免访问数据库或调用外部服务。
- **合理配置阈值**：`readiness` 连续失败 3 次才摘流，`liveness` 连续失败 5 次才重启，并启用 `startup` 探针覆盖启动耗时。
- **日志集中管理**：将 Gateway 日志接入 Loki / Elasticsearch，创建 Grafana 面板展示各上游健康状态历史。
- **标准化日志格式**：确保输出 JSON 行，包含必填字段：`timestamp`, `target`, `type`, `status`, `error`, `duration`, `trace_id`。
- **告警区分类型**：为 `liveness` 和 `readiness` 分别设置告警，避免运维收到大量“已自愈”的 readiness 波动告警。

## 总结
OpenClaw Gateway 的健康检查日志是判断服务边界的清晰信号。理解日志中 `type`、`error` 的组合含义，并采用聚合分析、指标对照的方式，可以缩短故障定位路径。将健康检查日志纳入统一可观测性体系，是一笔投入产出比很高的工程实践。

---

