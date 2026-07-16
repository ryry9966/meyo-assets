---
title: OpenClaw Gateway 健康检查日志排查指南：从零星告警到根因定位
feedId: 29364
source: 综合讨论
publishedAt: 2026-07-17
---

# 背景

在 OpenClaw 体系中，Gateway 是所有 Agent、MCP 工具、插件调用的统一入口。它不仅要承担流量路由，还要通过健康检查端点向 Kubernetes、负载均衡器或监控探针报告自身及上游依赖的状态。不少团队在配置了 `/healthz`、`/ready` 这类端点后，习惯性地“报绿即无事”，却忽略了日志里沉淀的大量诊断信息。当流量异常、Pod 被反复重启或上游服务偶发超时时，Gateway 的健康检查日志往往是离根因最近的第一手线索。

本文基于 OpenClaw Gateway v2.3 实践，梳理如何正确解读健康检查日志，避免被表面现象误导。

# 问题：日志就在那里，但你未必看得懂

典型的场景有几个：

1. **日志刷屏，关键信息被淹没**。Gateway 默认在 INFO 级别下，健康检查每分钟打印一行 `health check passed`，看起来一切正常；但当某个依赖开始抖动时，偶尔出现的 WARN 日志很快被后续的 PASS 刷掉，没人注意到。
2. **健康检查失败 ≠ 服务不可用**。`/ready` 失败会导致 Pod 被摘除，但失败原因可能是非关键依赖（如可选的缓存预热）超时，而实际 Agent 调用链路完全健康。
3. **日志只告诉你“挂了”，不告诉你“为什么”**。早期的 Gateway 版本在健康检查失败时只输出类似 `backend check failed` 的模糊信息，排查仍需去翻上游服务的日志。

要把健康检查日志用好，需要结合配置、日志级别和输出结构的调整，并形成一套固定的排查 SOP。

# 做法与步骤

## 1. 确认探针配置与日志输出端点

先检查 Gateway 的部署清单（如 K8s Deployment）中 `livenessProbe` 与 `readinessProbe` 分别指向哪个端点。在 OpenClaw Gateway 中，通常：

- `/healthz` 仅检查 Gateway 进程自身是否存活，返回状态码 200/500。
- `/ready` 会检查所有配置的依赖项，并返回结构化 JSON，适合用作 `readinessProbe`。

若两个探针指向同一端点，务必区分开。`/ready` 的响应时间可能较长（依赖项多时可达 2-3 秒），用在存活性探测上会引发超时重启。

## 2. 开启健康检查的调试日志

Gateway 的健康检查日志由 `gateway.health` 组件控制。在配置文件中增加：

```yaml
logging:
  level:
    gateway.health: debug
```

重新加载配置后，健康检查会在 DEBUG 级别下输出每次检查的详细耗时、上游地址、失败原因。如果担心日志量过大，可以在日志聚合系统中设置采样，而不是直接关掉。

## 3. 解读 `/ready` 返回的结构化日志

一次典型的 `/ready` 响应：

```json
{
  "status": "degraded",
  "checks": {
    "database": {"status": "ok", "duration_ms": 42},
    "cache": {"status": "ok", "duration_ms": 2},
    "plugin:mcp-tool": {"status": "failing", "error": "dial tcp 10.0.1.23:9090: i/o timeout"},
    "upstream:agent-executor": {"status": "ok", "duration_ms": 31}
  }
}
```

对应日志中的 DEBUG 条目会包含`checkName`、`status`、`error`等字段。排查时可以这样入手：

- 过滤 `"status":"failing"` 的日志行，看 `error` 字段是否稳定复现。
- 若 `error` 为 `i/o timeout` 或 `connection refused`，结合 `duration_ms` 判断是偶发还是持续；持续超时通常是网络策略或上游 Service Endpoint 变化。
- 若 `error` 中出现 `TLS handshake error`，先检查证书有效期，再检查 Gateway 是否信任自签 CA。

## 4. 建立故障关联

单靠健康检查日志还不够。已配置的可观测性栈中，Gateway 的 Metrics 会暴露 `openclaw_gateway_health_check_duration_seconds` 和 `openclaw_gateway_health_check_status`。当日志中出现 `plugin:mcp-tool` 持续失败时，可以立即关联该 MCP 工具调用的错误率，判断是否已影响实际业务。

# 踩坑点

**坑1：`/ready` 的依赖链过于严格**
业务团队曾把深度学习模型推理服务作为 Gateway 的强依赖加入 `/ready`，导致模型加载时 Gateway Pod 被摘除，引起全量断流。解法是将非必要依赖设为 `required: false`，并让 `/ready` 返回 `degraded` 而非 `unhealthy`。

**坑2：连接池耗尽导致健康检查超时**
Gateway 检查上游时复用了与业务流量相同的连接池。当业务瞬间并发打满时，健康检查也拿不到连接，最终超时触发重启。将健康检查使用的 HTTP 客户端配置独立连接池，或使用旁路端口，可彻底根治。

**坑3：kubelet 重试导致日志风暴**
Kubernetes 的 readiness probe 有 `periodSeconds` 与 `failureThreshold`，Gateway 在失败期间会反复打印同类错误。如果与本地日志滚动策略不合理配合，可能撑爆存储。建议在日志输出端做 dudup 或结构化日志中携带唯一错误 ID，并在聚合层分组。

# 可复用建议

- **日志格式统一为 JSON**：Gateway 配置 `logging.format: json`，所有健康检查日志带`component`、`check`、`error_code`字段，方便 Loki/ES 检索。
- **自定义检查脚本**：对复杂的插件，可在 `preStop` 或独立的 health 容器中执行 `/ready` 并进行二次逻辑判断，再决定是不是真的要踢掉 Pod。
- **健康检查独立端口**：避免业务流量与探测流量竞争资源，可以启用 Gateway 的管理端口（如 9091）供 kubelet 访问。
- **设置合理的初始延迟与间隔**：部署后给 Gateway 至少 `initialDelaySeconds: 10`，给上游依赖预热时间。

# 总结

不要只盯着探针的绿灯。健康检查日志是 Gateway 稳定性的仪表盘，它记录的是每一次“心跳”背后的细节。把日志级别调到合适粒度，理解 `/ready` 返回的结构，把失败条数、耗时变化当一回事，才能在问题扩展到用户之前完成自愈或人工介入。做好日志治理，下一场 on-call 你会感谢现在的自己。

---

