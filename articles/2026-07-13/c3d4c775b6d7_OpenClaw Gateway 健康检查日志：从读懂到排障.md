---
title: OpenClaw Gateway 健康检查日志：从读懂到排障
feedId: 28904
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景

在基于 OpenClaw Gateway 构建的 Agent、MCP 及插件自动化链路中，上游服务的健康状态直接决定了负载均衡、故障转移和服务发现的准确性。OpenClaw Gateway 内置了主动与被动健康检查机制，并在日志中留下了详尽的状态轨迹。然而，多数实践者只关注服务是否“绿了”，一旦出现偶发 5xx 或路由抖动，往往第一时间怀疑网络抖动或下游服务问题，却很少反向审视健康检查日志所给出的真实信号。

本文面向 OpenClaw 网关的日常运维与排障场景，厘清健康检查日志的关键字段、典型异常模式以及可复用的排障流程，帮助工程团队从“看个大概”进阶到“精准定位”。

## 问题：你以为的健康，可能只是假象

常见误区有三个：

1. **HTTP 200 就等于健康**：忽略了自定义健康检查端点（如 `/health` 返回 200 但体内 `status: "degraded"`），OpenClaw 虽然支持响应体断言，但很多人并未配置。
2. **把超时当作服务挂了**：主动健康检查的超时（`timeout_ms`）与上游连接超时、读取超时并非同一阈值，错误叠加时容易误判。
3. **日志级别开得太低**：默认 `info` 级别只记录状态变更，看不到连续探测的细节，导致偶发失败无迹可寻。

这些问题最终会表现为：网关剔除健康节点、负载不均、或者对故障实例恢复感知延迟，严重时会引发雪崩。

## 做法：从日志里挖出真相

### 1. 确认日志输出格式
OpenClaw Gateway 健康检查日志遵从统一的结构化输出（`json` 或 `logfmt`）。推荐在 `gateway.yaml` 中将健康检查相关 logger 调整为 `debug`：

```yaml
loggers:
  - name: gateway.health_checker
    level: debug
```

输出样例：
```json
{
  "ts": "2025-03-11T10:23:45.123Z",
  "logger": "gateway.health_checker",
  "level": "debug",
  "msg": "active check completed",
  "upstream": "mcp-tool-service",
  "instance": "10.2.1.12:9080",
  "check_type": "http",
  "status": "unhealthy",
  "status_code": 503,
  "latency_ms": 1201,
  "error": "context deadline exceeded",
  "consecutive_failures": 3,
  "healthy_threshold": 2,
  "unhealthy_threshold": 3
}
```

### 2. 关键字段解读
- **check_type**：`http`、`tcp`、`grpc`，不同协议健康检查的超时与断言不同。
- **status**：`healthy` / `unhealthy`，是综合判定结果，不要只依赖 `status_code`。
- **consecutive_failures** & **unhealthy_threshold**：当前连续失败次数达到 unhealthy_threshold 时才会从负载均衡中剔除，这解释了为什么服务已返回 503，但网关仍转发了一小段时间。
- **latency_ms**：若接近或超过配置的 `timeout_ms`（默认 1000ms），说明服务响应慢，需考虑调高阈值或优化下游。
- **error**：包含具体错误信息，如 `connection refused`、`context deadline exceeded`、`tls handshake timeout` 等，直接指向根因。

### 3. 快速过滤与统计
使用 `jq` 进行实时分析：
```bash
# 查看某个上游的所有不健康事件
tail -f gateway.log | jq 'select(.logger=="gateway.health_checker" and .status=="unhealthy")'

# 按实例统计失败次数
grep "gateway.health_checker" gateway.log | jq -r '[.instance, .status] | @tsv' | awk '{cnt[$1]++} END{for(i in cnt) print i, cnt[i]}'
```

### 4. 排障决策树
当出现不健康事件时，按以下顺序排查：
1. **检查 error 字段**：如果是连接拒绝 → 实例未启动或端口错误；TLS 握手超时 → 证书或网络策略问题。
2. **观察 latency_ms 趋势**：若逐步上升，通常是下游资源饱和（如线程池满、数据库慢查询）。
3. **检查 consecutive_failures 是否达到阈值**：若未达到，网关不会立即摘除，短暂波动可容忍；若持续处于阈值边缘，说明探测不稳定，需检查健康检查端点逻辑。
4. **检查主动与被动健康检查是否冲突**：被动检查（通过实际请求流量判定）会标记“pending”状态，可能导致与主动检查状态不一致，需统一判定策略。

## 踩坑点
- **健康检查端点不做幂等性保护**：某些 `/health` 接口可能触发写操作或缓存刷新，高频探测会导致性能问题。
- **改完配置忘记重载**：OpenClaw 支持动态重载，但 `unhealthy_threshold` 等参数的修改需要通过 Admin API 或信号重载生效，直接改文件不自动加载。
- **日志输出过多导致性能下降**：`debug` 级别仅在排障时开启，日常使用 `info` 并配合 Metrics（如 `openclaw_health_check_failures_total`）做监控。

## 可复用建议
1. **建立统一监控面板**：将健康检查状态指标（Prometheus 格式暴露）绘制成时序图，设定 `consecutive_failures >= unhealthy_threshold - 1` 为预警线。
2. **告警规则双保险**：既要基于网关 Metrics 告警，也要从日志中聚合 `"status":"unhealthy"` 事件，防止指标上报延迟。
3. **记录健康检查变更审计**：在 `info` 级别日志中必定会打印状态变更（`healthy → unhealthy` 及其反向），建议将此事件接入事件总线，用于自动化根因分析。
4. **自定义健康检查断言**：充分利用 OpenClaw 的 `expected_body` 或 `expected_status` 字段，例如检查 JSON 中的 `{"status":"ok"}`，避免假健康。

## 总结
OpenClaw Gateway 的健康检查日志不是“有事才看”的黑盒，而是服务韧性的诊断利器。通过开启合适的日志级别、理解状态机阈值和关键字段，工程团队可以在几行命令内完成从现象到根因的定位。将日志、Metrics 和告警联动起来，才是真正面向生产环境的健康管理方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/b9a55f1e9403fcdf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/bc987635294bfae0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/0059e7329434ccf4.png)

