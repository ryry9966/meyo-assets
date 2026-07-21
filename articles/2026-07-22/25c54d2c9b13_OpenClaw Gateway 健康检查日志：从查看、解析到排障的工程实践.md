---
title: OpenClaw Gateway 健康检查日志：从查看、解析到排障的工程实践
feedId: 29994
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

OpenClaw Gateway 通常作为 Agent、MCP 服务器与各类插件之间的统一入口，承载请求路由、认证和协议转换。在生产环境中，Gateway 的健康状态不仅由 HTTP 状态码决定，更取决于它能否可靠地连接到后端依赖 —— 比如 Redis、消息队列、MCP 服务端点或嵌入式规则引擎。无论是 Kubernetes 的探针，还是负载均衡器的健康检查，当 Gateway 返回 503 时，只看到“服务不可用”远远不够，你需要知道**到底是哪一个依赖出了问题**。

健康检查日志就是这条排障路径的入口。但在实际工程中，许多配置默认只输出简单的 `GET /health 200` 行，甚至把 liveness 和 readiness 的日志混在一起，让排障变成猜谜。本文梳理一套可复用的日志配置与解读方法，帮助你在自动化流水线出现抖动时，迅速定位根因。

## 问题：模糊的不可用，混乱的日志

典型症状：
- 监控面板上 readiness 探针间歇性失败，但 Gateway 进程并未重启；
- 容器日志里只有重复的 `health check returned 503`，没有指明依赖详情；
- 日志时间戳缺少时区，导致跨地域排障时无法对齐上下游服务的异常时间；
- 健康检查日志与 access log 混杂，关键时刻被请求日志淹没。

这些问题的共同根源是：**健康检查的日志粒度不足，格式非结构化，且未按探针类型分流**。

## 做法 / 步骤

### 1. 启用详细健康检查日志

在 OpenClaw Gateway 配置文件（通常是 `openclaw-gateway.yaml`）中调整日志模块级别：

```yaml
logging:
  level: info
  modules:
    health: debug          # 输出依赖检查详情
    health.probe: trace    # 可选，输出每次探针的原始结果
```

如果通过环境变量控制，可设置：
```
HEALTH_LOG_LEVEL=debug
```

重启后，readiness 日志会从简单的状态行变成包含 `checks` 数组的结构化记录。

### 2. 使用结构化格式与探针标签分离

要求日志输出为 JSON，并保证每条健康检查记录带有 `probe` 字段：

```yaml
logging:
  format: json
```

正常 JSON 记录示例：
```json
{
  "timestamp": "2025-03-15T09:12:33.421Z",
  "level": "debug",
  "logger": "health.check",
  "probe": "readiness",
  "status": "failing",
  "checks": [
    {"name": "redis", "state": "passing", "duration_ms": 1.2},
    {"name": "mcp:knowledge", "state": "failing", "error": "dial tcp 10.0.1.5:8088: i/o timeout"}
  ],
  "message": "readiness check completed"
}
```

其中 `logger` 固定为 `health.check`，`probe` 取值 `liveness` 或 `readiness`，这些字段成为过滤的关键。

### 3. 按探针类型查看日志

在 Kubernetes 环境中：
```bash
# 仅看 readiness 失败记录
kubectl logs -l app=openclaw-gateway --tail=200 -f | grep '"probe":"readiness"' | grep '"status":"failing"'
```

Docker 环境：
```bash
docker logs --tail 500 openclaw-gateway 2>&1 | grep '"probe":"readiness"' | jq 'select(.status=="failing")'
```

使用 `jq` 可以更精细地提取失败依赖：
```bash
... | jq 'select(.status=="failing") | .checks[] | select(.state=="failing") | {name, error}'
```

### 4. 解读依赖检查失败的模式

一次 readiness 失败日志通常会给出明确的失败依赖名和错误信息。常见模式：
- `dial tcp ... i/o timeout`：MCP 服务不可达或网络策略阻断；
- `connection refused`：目标端口未监听；
- `NOAUTH Authentication required`：Redis 密码错误；
- `context deadline exceeded`：单个依赖响应超时，需调大健康检查的超时参数。

结合 Gateway 的 metrics 端点（如 `/metrics`）中的 `openclaw_health_check_duration_seconds` 和 `openclaw_health_check_failures_total`，可以进一步量化故障范围和频率。

### 5. 分离健康检查日志流

避免与 access log 混合的最佳实践是使用日志路由。例如，通过 stdout/stderr 区分：让 access log 输出到 stdout，健康检查细节输出到 stderr，或在文件侧使用不同的 appender：

```yaml
logging:
  appenders:
    access:
      type: console
      stdout: true
    health:
      type: file
      path: /var/log/openclaw/health.log
```

然后只将 `health.check` logger 指向该 appender。这样既保持了主日志清爽，又便于采集和分析。

## 踩坑点

- **debug 日志洪流**：在生产环境开启 `health: debug` 会显著增加日志量，尤其在探针间隔较短（<10秒）时。建议只在排障时动态开启，或通过 `health.probe: debug` 仅记录详细结果而不输出每次探针的开始/结束事件。
- **时间戳时区缺失**：如果 JSON 时间戳缺少时区信息（如 `2025-03-15T09:12:33`），与上游服务日志对时间时会很痛苦。要求日志组件强制输出 UTC 或带偏移量的 ISO8601 格式。
- **ready 与 live 混淆**：liveness 探针不应包含复杂依赖检查，否则依赖抖动会误杀 Pod。但某些初学者配置中将两者共用同一套检查逻辑，导致 liveness 日志也出现依赖错误。务必在 Gateway 配置中将 `liveness` 探针的路由指向仅检查进程存活的简单端点。

## 可复用建议

1. **构建 health-log 解析脚本**：一个简单的 `jq` 管线可以定时提取最近 5 分钟的失败检查，输出为 Markdown 表格，发送到值班频道。
2. **配置告警规则**：在 Prometheus Alertmanager 中设置规则：`openclaw_health_check_failures_total{probe="readiness"} > 3` 持续 5 分钟，通知对应插件的负责人。
3. **集成到集中日志平台**：将 Gateway 的健康日志流单独索引，创建仪表盘，展示各依赖的实时健康状态，以及每次故障的持续时间。
4. **在自动化流水线中注入排障信息**：当 CI/CD 部署新插件后，强制检查一次 Gateway 的 readiness 日志，若出现新的 `failing` 依赖，则回滚并贴出错误详情。

## 总结

OpenClaw Gateway 的健康检查日志远不止一行 HTTP 状态码。通过开启适度详细日志、强制 JSON 结构、区分探针类型并分离日志流，你可以让每一次 503 背后的依赖故障暴露无遗。在插件数量增多、MCP 连接复杂化的演进过程中，这套日志实践能够将平均排障时间从分钟级降低到秒级，让自动化流水线真正具备可观测性。

**记住**：你看到的健康检查日志，就是 Gateway 在替所有下游服务向你汇报的体检报告。读懂它，你才能知道整个系统是真的健康，还是在勉强支撑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/f0b2bfb217076d83.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/64113fc5fd3bec46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/47833214af7e2b4d.png)

