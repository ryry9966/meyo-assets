---
title: OpenClaw Gateway 健康检查日志解读与排障指南
feedId: 29115
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在基于 OpenClaw 搭建的 Agent / MCP 服务体系中，Gateway 承担着流量入口、协议转换与健康检查聚合的角色。它的健康检查日志是判断下游 MCP 服务器、插件或自定义 Agent 是否就绪的最直接信号。许多自动化流水线（如滚动更新、自愈重启、负载摘除）都依赖这些日志触发动作。然而，健康检查日志往往被当成“黑盒输出”，出现问题后要么全量忽略，要么盲目抓取全文告警，浪费排查时间。

本文从日常运维视角出发，梳理 OpenClaw Gateway 健康检查日志的结构、典型异常模式与排障方法，并给出生产环境可复用的观察建议。

## 常见问题场景

1. **误报“不健康”**：Gateway 间歇性将正常服务标记为 `UNHEALTHY`，但手动探测完全正常。日志中常伴随 `probe_timeout` 或 `connection_reset` 字样。
2. **健康检查完全丢失**：某个 MCP 服务器未被 Gateway 例行检查，日志中看不到对应 target 的任何记录，导致服务挂掉后毫无感知。
3. **状态翻转抖动**：日志中同一 target 在 `HEALTHY` 与 `UNHEALTHY` 之间高频切换，每次持续几秒，引发大量告警与摘除/恢复动作。
4. **假健康**：日志显示 `HEALTHY`，但实际请求全部失败。通常是因为健康检查端点仅探测端口存活，未覆盖业务逻辑。

## 日志结构解读

OpenClaw Gateway 的健康检查日志统一输出为结构化 JSON，每行一条记录，关键字段如下（版本 v0.7+）：

```json
{
  "ts": "2025-04-01T14:22:10.231Z",
  "level": "info",
  "msg": "health check result",
  "target": "mcp-server-01",
  "addr": "10.0.2.15:9090",
  "probe_type": "http",
  "status": "HEALTHY",
  "latency_ms": 12.3,
  "fail_reason": "",
  "consecutive_failures": 0
}
```

`target` 对应 OpenClaw 中注册的服务名；`probe_type` 默认为 `http`，也支持 `tcp` 和 `grpc`；`consecutive_failures` 是 Gateway 内部计数器，触发摘除的阈值默认为 3，可在配置中调整。`fail_reason` 为诊断提供了第一条线索：`context deadline exceeded` 表示超时，`connection refused` 说明端口未监听，`status code 503` 则说明服务能连通但主动拒绝。

## 排障步骤

### 1. 首先确认日志是否产生
在 Gateway 容器或宿主机上执行：
```
journalctl -u openclaw-gateway -f | grep "health check"
```
若无任何输出，检查配置文件 `probes` 段落是否为空，以及目标服务是否在 `targets` 列表中正确注册。一个常见疏忽是：新增 MCP 服务器后只更新了路由，却忘记将其加入健康检查列表。

### 2. 分析单条异常记录
挑出一条 `status: "UNHEALTHY"` 日志，优先看 `fail_reason` 和 `latency_ms`：
- 超时类错误（`timeout`）：先确认探活 timeout 设置（默认 500ms）是否比服务实际启动耗时要短。冷启动时，某些 MCP 推理 Agent 需加载模型，首次响应可能超过 2s，此时应增大 `probe_timeout` 或使用 `initial_delay_seconds` 类参数。
- 连接重置（`connection_reset`）：常见于服务端频繁重启或连接池耗尽。可结合 Gateway 的 `keepalive` 配置与服务端 `MaxConcurrentStreams` 对比。
- 状态码异常（`status code 503`）：服务过载或正在关闭。查看服务自身健康检查端点是否区分“存活”与“就绪”，建议 Gateway 使用就绪探针，避免流量打到尚未准备好的实例。

### 3. 抖动分析
使用简单命令统计时间窗口内的翻转次数：
```
cat /var/log/openclaw/health.log | jq 'select(.ts >= "2025-04-01T14:00:00Z") | select(.target=="mcp-server-01") | .status' | uniq -c
```
若 1 分钟内状态变化超过 5 次，通常是网络抖动或健康检查间隔（`interval`）设置过短。可将 `interval` 从 5s 放宽至 10s，同时增加 `failure_threshold`，避免瞬时波动触发连锁反应。

### 4. 假健康排查
对 `status: "HEALTHY"` 但请求大量失败的目标，需检查健康检查端点具体返回的内容。OpenClaw Gateway 的 HTTP 探活默认只校验 2xx 状态码，但很多服务会单独提供 `/healthz` 返回 200 却内部挂起。建议在 Gateway 配置中启用 `body_regex` 选项，匹配预期关键字（如 `"status":"pass"`），或在服务侧实现深度探针。

## 踩坑点

- **时区不一致**：Gateway 使用 UTC，但运维人员习惯本地时间检索日志。若未统一时区，容易漏看故障发生时段，务必在 Grafana 或日志平台按 UTC 展示。
- **DNS 缓存干扰**：如果 Gateway 通过域名探活，DNS 解析失败会被记录为 `lookup failed`，但可能被误判为服务宕机。应确保 Gateway 所在容器的 DNS 配置可靠，并结合 `hostAliases` 固化核心服务 IP。
- **TLS 证书过期**：gRPC 探活依赖双向 TLS 时，证书过期会导致 `UNHEALTHY`，但日志 `fail_reason` 可能仅显示 `transport error`，需配合系统时间与证书有效期排查。
- **日志量爆炸**：高频全量日志可能撑满磁盘。建议将 `level` 为 `info` 的健康检查日志与错误日志分离，仅保留状态变化事件，或使用采样策略。

## 可复用建议

1. **标准化探活端点**：要求所有接入 OpenClaw 的 MCP 服务器暴露 `/.well-known/openclaw/ready`，并返回固定 JSON 结构 `{"status":"ready","dependencies":["db","cache"]}`，便于 Gateway 解读。
2. **分组告警**：将同一服务组的抖动合并为单条告警，延迟 30s 发出，减少噪声。
3. **利用 metrics 关联**：Gateway 自身暴露 Prometheus metrics（如 `openclaw_health_check_failures_total` 和 `openclaw_health_check_duration_seconds`），可将日志与 metrics 叠加看板，快速定位是普遍延迟还是单点故障。
4. **预热宽限期**：在部署流水线中，先启动服务并等待 Ready，再将其加入 Gateway 目标列表，避免冷启动误判。
5. **定期演练**：每季度模拟一次假健康场景，验证摘除逻辑和通知链路是否有效。

## 总结

OpenClaw Gateway 的健康检查日志并非简单的“正常/异常”二值信号，它包含连接状态、延时趋势和故障原因等工程化信息。遇到问题时，不要只盯着 `status` 字段，要结合 `fail_reason`、`consecutive_failures` 以及服务端的等效探针去交叉验证。制定符合业务容忍度的阈值、合理灰度日志输出，并建立与监控体系的联动，才能让这套日志真正从“记录”变为“可诊断的信号”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/52fe6e4ee15efde1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/a3b6c7a52c3d949f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/42d276e2e53597cc.png)

