---
title: OpenClaw Gateway 健康检查日志怎么看
feedId: 29292
source: 综合讨论
publishedAt: 2026-07-16
---

# OpenClaw Gateway 健康检查日志怎么看

在 OpenClaw 的运维实践中，Gateway 作为前端流量入口与 Agent 调度中心，它的健康检查（/healthz、/readyz）几乎是所有告警链路的第一跳。但很多同学习惯看监控面板上的红绿状态，一旦出现间歇性 503 或 Pod 反复重启，才回头翻日志，往往被满屏的请求 trace 挡住关键信息。本文梳理一套分析方法，帮助你在故障时快速从健康检查日志里提取有效信号。

## 背景问题

OpenClaw Gateway 使用标准 Kubernetes 探针，同时内部叠加了多层依赖检查：数据库连接、缓存、MCP 后端服务、插件运行状态等。健康检查端点会根据这些组件的实时状态聚合出最终响应码。一个典型的故障场景：监控显示 Pod Ready，但部分请求返回 503。查看事件只有“Liveness probe succeeded”，可实际服务已经不健康了。根因在于就绪探测配置的 `failureThreshold` 过高，且日志里只记录了“probe failed: context deadline exceeded”，没有展开哪个子检查超时，导致排查方向错误。

## 日志长什么样

Gateway 默认以结构化 JSON 输出日志。一条完整的健康检查记录可能类似：

```json
{
  "timestamp": "2025-04-09T10:23:47.123Z",
  "level": "info",
  "component": "healthz",
  "handler": "/readyz",
  "status": 503,
  "latency_ms": 5010,
  "checks": [
    {"name": "db", "state": "healthy", "latency_ms": 2},
    {"name": "mcp_backend", "state": "unhealthy", "error": "context deadline exceeded"},
    {"name": "plugin_registry", "state": "healthy", "latency_ms": 1}
  ]
}
```

当整体状态为不健康时，`level` 会变为 `warn` 或 `error`。理解这个结构是排查的起点。

## 实操步骤

### 1. 打开调试开关（临时）

生产环境为了性能通常只输出 `info` 以上级别，健康检查成功日志可能被抑制。故障复现期间，可以动态调高 Gateway 的日志级别：

```bash
export LOG_LEVEL=debug
# 或通过 admin API
curl -X PUT http://gateway:9090/admin/log-level -d '{"level":"debug"}'
```

但**务必设置生效窗口**，比如 5 分钟后自动恢复，避免日志量爆炸。

### 2. 精准过滤

使用 `jq` 过滤健康检查相关日志，只保留最近 10 分钟的记录：

```bash
kubectl logs deploy/openclaw-gateway --since=10m | \
  jq 'select(.component=="healthz" or .msg | startswith("probe"))' > health-debug.log
```

如果日志量巨大，可以只抓取非 200 状态的条目：

```bash
jq 'select(.component=="healthz" and .status >= 400)'
```

### 3. 解读 checks 数组

健康检查的成败通常写在 `checks` 里。对于 MCP 后端这类外部依赖，建议重点关注它的 `state` 和 `error`。如果看到 `dial tcp: connect: connection refused`，说明目标服务端口没起来；如果看到 `context deadline exceeded`，可能是后端响应过慢，超过了探针的 `timeoutSeconds`。此时不要只看 Gateway 自己的超时配置，还要对比 MCP 服务的实际处理延迟。

### 4. 关联探针配置

将日志中的连续失败次数与 Kubernetes 探针参数比对。假设 readinessProbe 配置为 `periodSeconds: 5`、`failureThreshold: 3`，那么看到 3 条间隔 5 秒的 503 就绪检查后，Pod 才会被摘除流量。如果日志中只有 2 条失败，但服务已经不可用，说明你的阈值设得过于宽松。根据日志的实际失败模式，调小 `failureThreshold` 或缩短 `periodSeconds`，并重新部署。

### 5. 时间线反推

利用时间戳还原事件序列。常见陷阱：Gateway 自身的超时设置比客户端或 Ingress 更宽松，导致日志记录到“probe failed”时，上游早已超时断开。应对齐 timeout 链条：Gateway → 健康检查探针 → 依赖服务，同时检查 `latency_ms` 是否已经触达阈值。

## 踩坑点与工程建议

- **日志抑制反噬**：曾有一个案例，运维同学为了提升性能，禁止了所有健康检查成功日志。结果 MCP 后端间歇性超时，网关就绪状态反复翻转，查了 4 小时找不到根因。最后临时开启 debug 日志才发现是 MCP 的连接池耗尽。**建议保留 error/warning 级别的健康检查日志，并至少抽样输出 1% 的成功日志**，方便推算探针延迟的 P99。

- **健康检查本身成为攻击面**：如果健康检查端点不做鉴权，且暴露了依赖细节（如 checks 里的具体错误），会增加信息泄漏风险。**生产环境可以只暴露聚合状态**，详细 checks 仅记录在日志内部，且日志访问权限收紧。

- **将健康检查日志接入告警**：不要只依赖 Kubernetes 的 Pod 状态告警。用 LogQL 或 Loki 规则，当 5 分钟内 readiness 检查出现 ≥3 次 503 且 `mcp_backend` unhealthy 时，直接触发通知。这样可以比 Pod 重启告警提前 3~5 分钟发现问题。

- **针对插件做分级检查**：对于非关键插件，Gateway 可以启用“软健康”模式，插件失败不导致整体 Ready 置为 False，只在日志里打 warning。这样可以避免一个非核心自动化插件拖垮整个集群。

## 可复用查询模板

建议团队内部沉淀几个 `jq` 或 LogQL 片段：

- 提取最近 10 次就绪检查的延迟分布：
  ```bash
  cat health.log | jq '[.latency_ms] | sort | {min: .[0], max: .[-1], med: .[length/2|floor]}'
  ```
- 统计各 checks 子项的不健康占比：
  ```bash
  jq -r '.checks[] | select(.state=="unhealthy") | .name' health.log | sort | uniq -c | sort -rn
  ```

把这些查询做成脚本或 Grafana 面板，可以极大缩短故障定位时间。

## 总结

Gateway 的健康检查日志是分布式系统健康状况的“脉搏图”。它不应只是运维排障的最后手段，而应该被主动消费、关联分析和规则化告警。理解日志结构、掌握过滤技巧，并结合探针配置做时间线推导，会让你在面对“Pod 是 Ready 的，但服务就是不可用”这类模糊故障时，把根因锁定得又快又准。健康检查日志从不是多余的输出，它是 Agent 与 MCP 生态里最诚实的状态汇报。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/8241a4eced5ac02a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/17297d48dd821ed2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/cad4dfa696aa2de2.png)

