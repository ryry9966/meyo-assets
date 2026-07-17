---
title: OpenClaw Gateway 健康检查日志排障指南
feedId: 29401
source: 综合讨论
publishedAt: 2026-07-17
---

# OpenClaw Gateway 健康检查日志排障指南

## 背景

在基于 OpenClaw Gateway 搭建的 Agent / MCP 接入层里，健康检查几乎是每一条流量链路的第一个环节。无论是负载均衡器的存活探测，还是 MCP Server 注册时的就绪检查，都会反复触发网关的健康检查端点。对应的日志在 `/var/log/openclaw/health.log`（或 systemd journal）中频繁出现，尤其在多实例部署、短间隔探活时，很容易淹没其他有效信息。久而久之，这条日志就从“帮手”变成了“噪音”。

然而，健康检查日志恰恰是判断上游服务连通性、网关自身状态和证书配置是否正确的第一手证据。理解这些日志的结构和陷阱，是运维 OpenClaw 网关绕不开的基本功。

## 问题

多数配置里，OpenClaw Gateway 会暴露 `/healthz` 和 `/readyz` 两个端点（可在 `gateway.yaml` 中自定义）。探活请求通常由外部系统（如 Kubernetes、HAProxy、Consul）发起，频率可能在 2~5 秒。如果没有合理的日志策略，几个小时后 `journalctl -u openclaw-gateway | grep health` 会刷出数十万条记录，让真正的异常淹没其中。

常见困扰包括：

- 日志量巨大，难以快速筛选失败探活。
- 健康检查失败时，错误信息含糊（如 `upstream_unhealthy`），无法直接定位根因。
- 探活请求被错误地转发到上游服务，导致上游日志也受污染。
- 连续失败后，网关被摘除，却找不到首次失败的那条日志。

## 做法 / 步骤

### 1. 理解日志字段

OpenClaw Gateway 的健康检查日志默认采用结构化 JSON，形如：

```json
{
  "timestamp": "2025-03-23T14:02:01.123Z",
  "level": "info",
  "component": "healthcheck",
  "check": "readiness",
  "result": "ok",
  "upstream": "mcp-server-1",
  "latency_ms": 3,
  "status_code": 200,
  "error": ""
}
```

核心字段涵义：

- `check`: `liveness` 或 `readiness`，取决于调用端点。
- `result`: `ok`、`degraded`、`failed`。`degraded` 常见于某个上游未就绪但网关整体仍可服务。
- `upstream`: 受检上游服务名（如果探活会级联检查后端）。
- `status_code`: 上游返回的 HTTP 状态码，仅在级联探活时填充。
- `error`: 错误详情，仅失败时非空。

### 2. 快速过滤失败记录

在日志量陡增的现场，直接 `grep` 并不高效。推荐使用 `jq` 做结构化过滤：

```bash
# 过滤出所有失败的健康检查
journalctl -u openclaw-gateway -o json --since "10 minutes ago" \
  | jq -r 'select(.MESSAGE | fromjson.component == "healthcheck" and fromjson.result == "failed")'
```

若日志落盘，可以直接解析文件：

```bash
cat /var/log/openclaw/health.log \
  | jq 'select(.result == "failed")' -c
```

### 3. 聚合与趋势分析

要判断是偶发故障还是持续恶化，可以按时间窗口聚合：

```bash
# 统计每分钟失败次数
cat health.log \
  | jq -r 'select(.result == "failed") | .timestamp[0:16]' \
  | sort | uniq -c
```

如果失败集中在某个上游，进一步按 `upstream` 分组，快速锁定问题服务。

### 4. 降低日志噪音

健康检查成功记录在稳定运行时价值有限，可以只记录状态变更或失败。在 `gateway.yaml` 中配置：

```yaml
health:
  log_level: change_only   # 默认是 all
```

这样，只有状态从 `ok` 切换到 `failed` 或反向时才会输出日志，日志量可降低 90% 以上。

如果必须保留全量日志，建议将健康检查日志分流到独立的文件或 syslog facility，通过 `rsyslog` 或 `Vector` 单独处理，避免污染主日志流。

### 5. 增加诊断信息

在排障时临时提高日志详细程度（需重启，慎重在生产期操作）：

```yaml
health:
  log_level: debug  # 会输出 TCP 握手时间、TLS 协商细节等
```

`debug` 级日志会打印连接复用的状态，对于定位间歇性的 `connection refused` 或 TLS 证书过期非常有用，但会明显增加 I/O 压力，排障完毕务必恢复。

## 踩坑点

- **级联探活的超时盲区**  
  网关自身探活超时默认为 2 秒，但上游服务的就绪检查可能需要更久（例如加载模型）。如果未在 `upstream.health.timeout` 单独调大，会出现“网关认为上游挂了，但上游其实只是慢”的情况，日志中 `error` 字段会记录 `context deadline exceeded`。

- **TLS 证书错误被泛化**  
  当上游启用 mTLS 但网关证书过期时，健康检查日志可能只显示 `upstream_unhealthy`，而不会直接暴露 `x509: certificate has expired`。需要打开 `debug` 级别或检查网关自身的 TLS 错误日志（通常在独立文件）。

- **健康检查端点误转发**  
  很多人会在网关路由里配置 `catch-all` 规则，导致 `/healthz` 请求被转发给上游。正确做法是保证健康检查端点直接在网关本地处理，不经过路由链。检查 `routes` 配置确认是否对 `health` 路径做了 `bypass` 或 `internal` 标记。

- **日志轮转不及时**  
  高频探活会产生大量日志，如果未配置 `logrotate` 或 `maxsize`，容易撑满磁盘。建议按大小或小时轮转，保留 2 小时即可。

## 可复用建议

1. **设置健康检查告警**  
   在集中日志系统（如 Loki/Grafana）中创建指标：`rate of healthcheck failed > 0 for 5m`，并关联上游服务名。这样可以先于负载均衡器发现后端异常。

2. **编写解析脚本**  
   将上面的 `jq` 命令封装为脚本 `openclaw-health-analyze`，按失败次数、上游、时间范围输出报告，减少排障时的手动拼凑。

3. **单独监控探活端点**  
   提供 `/healthz` 和 `/readyz` 之外的一个轻量级 `/ping`，不在日志中记录，纯粹给外部系统高频探测，进一步解耦监控与排障日志。

4. **利用分布式追踪补充**  
   健康检查只是“点状”信息，当出现 `degraded` 时，结合 OpenTelemetry trace 查看上游调用链，可以判断是网络、证书还是业务逻辑导致的拒绝服务。

## 总结

OpenClaw Gateway 的健康检查日志像一座金矿，但需要正确的工具和方法才能提炼出价值。默认配置适合开发环境，落地到生产时必须针对日志量、超时和错误可见性做主动调优。记住：**只记录状态变化，失败时保留细节，定位靠聚合而非单条**，可以让这套日志从噪声源变回可靠的探针。下次再看到刷屏的 `ok`，不妨先调为 `change_only`，然后把注意力留给真正的失败与超时。

---

