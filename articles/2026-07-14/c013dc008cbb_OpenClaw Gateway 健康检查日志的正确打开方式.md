---
title: OpenClaw Gateway 健康检查日志的正确打开方式
feedId: 29101
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景：为什么健康检查日志值得单独看

在 OpenClaw 生产部署中，Gateway 通常位于接入层，承担流量代理、协议转换、MCP 工具授权等功能。Kubernetes 或负载均衡器会定期对 Gateway 发起 HTTP/TCP 健康探针（liveness/readiness），这类请求的频率远高于业务流量，但排查问题时经常被忽略。

健康检查日志的价值在于：
- 探针失败会直接触发 Pod 重启或摘流，比业务 500 更致命
- 探针成功与否反映 Gateway 内部状态（线程池、连接池、依赖服务可用性）
- 大量探针日志可能淹没存储，需要合理的日志级别和采样策略

本文聚焦 OpenClaw Gateway 的健康检查日志，给出工程化的查看方法、常见问题定位路径，以及可复用的日志治理建议。

## 问题现象

- 容器反复重启，events 显示 Liveness probe failed，但 kubectl logs 中很难找到对应时刻的异常
- 日志被打满 “GET /healthz 200” 或 “OK”，真正错误被滚动覆盖
- 健康检查返回 200，但实际已无法处理业务请求（半死状态）
- Gateway 在插件更新或 MCP 后端切换后探针偶发超时，但本地复现困难

## 做法：三件事看清健康检查日志

### 1. 确认日志输出位置与格式

OpenClaw Gateway 默认使用结构化日志（JSON 或 logfmt），健康检查请求同样会落入 access log。日志级别通常为 `info`，但可通过环境变量调整：

```bash
# 提高健康检查日志级别以减少噪声（单独关闭）
export LOG_LEVEL_HEALTH=warn
# 或完全静默，但不建议生产环境直接关闭
```

使用 kubectl 或 docker logs 收集一段时间的完整日志，按时间过滤：

```bash
kubectl logs deploy/openclaw-gateway --since=5m | grep -E 'health|/healthz|/readyz'
```

如果你的 Gateway 前面有 Envoy/NGINX 再做一层代理，注意区分「Gateway 自身记录的探针日志」和「上游代理记录的探针日志」，两者时间戳可能不一致。

### 2. 结构化日志字段解读

典型一条 Gateway 健康检查日志会包含：

```json
{
  "time": "2025-03-28T10:23:01.456Z",
  "level": "info",
  "msg": "request completed",
  "method": "GET",
  "path": "/healthz",
  "status": 200,
  "latency_ms": 12,
  "client_ip": "10.244.1.23",
  "upstream": "",
  "probe_type": "liveness",
  "error": ""
}
```

关键字段：
- `latency_ms`：单次探针耗时。如果接近或超过探针配置的 `timeoutSeconds`，说明 Gateway 内部处理变慢，常见原因包括线程池耗尽、插件加载慢、下游 MCP 服务未及时关闭连接。
- `client_ip`：可以区分是来自 kubelet 的探针还是外部监控系统。kubelet 探针源 IP 通常是节点 Pod CIDR，变化时可辅助判断节点故障。
- `error`：如果非空，即使 status=200 也可能是降级响应，需要结合 Gateway 代码查看降级逻辑。
- `probe_type`：新版 OpenClaw Gateway 支持在日志中注入探针类型（liveness/readiness/startup），有助于区分不同探针的行为。

### 3. 主动触发问题复现

不要等探针失败才去看日志。可以在本地或在非生产环境通过压测探针接口模拟异常：

```bash
# 并发探测，观察 Gateway 线程模型表现
hey -z 30s -c 50 http://localhost:8080/healthz
```

同时调整日志级别到 `debug`，观察健康检查处理过程中是否有资源获取超时（如数据库连接池、MCP Server 健康检查级联）。

## 踩坑点

- **静默吞掉错误**：一些定制版 Gateway 为了“优雅”，任何错误都返回 200，同时只打一条 warn。这会导致 kubelet 永远认为 Pod 健康，直到 OOM 或死锁。建议在 readiness 探针中对关键依赖做真实检查，并显式返回 503。
- **日志异步丢失**：进程崩溃前的最后一批日志可能来不及刷盘。如果探针失败后容器立即重启，kubectl logs --previous 可能也看不到失败时刻的日志。解决方案：增加日志缓冲刷盘频率，或在探针 handler 中同步写入 critical 级别日志。
- **把健康检查日志完全关闭**：某些教程建议关闭减少存储成本，实际上会牺牲排障能力。正确做法是保留 warning 以上，或对 health 端点单独设置 log filter，只记录非 200 的请求。
- **忽略网关已死但探针未死**：这种情况常见于 Gateway 的 event loop 被阻塞，但 HTTP health handler 仍能快速响应（因为它在独立线程或仅检查进程存活）。应确保探针至少检查一次内部任务队列深度或最近成功请求时间。

## 可复用建议

1. **日志级别分层**：将健康检查日志独立配置，默认只记录非 200 状态码或延迟超过阈值（如 500ms）的探针，减少噪声。
2. **探针日志与业务日志分离输出**：将健康检查日志打到单独的 stdout/stderr stream 或文件，方便采集和告警。
3. **使用 Prometheus 指标替代部分日志排查**：`oc_gateway_healthcheck_duration_seconds` 和 `oc_gateway_healthcheck_failures_total` 等指标更易设置告警阈值，日志仅做现场还原。
4. **全链路探针设计**：实现 `/healthz?deep=true` 或独立 `/readyz/deep`，级联检查 MCP Server、插件状态、许可证状态，避免简单返回 200 的假健康。
5. **在 CI 中加入探针日志断言**：部署后自动运行探针并检查日志中不出现 error 级别记录，延迟超过基线则报警。

## 总结

OpenClaw Gateway 的健康检查日志不应被视为背景噪声，而是系统自检的第一手信号。通过明确日志字段含义、合理配置采样和存储、在探针逻辑中做深度检查，可以提前发现线程池压力、依赖超时等深层次问题。踩坑经历表明，单纯依赖 HTTP 200 是危险的，需要结合结构化日志和指标体系，才能在生产环境中睡个安稳觉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/6e6665e3bb34bc61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/e67a9cb99338a8c4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/c0c5dc2b45f27d69.png)

