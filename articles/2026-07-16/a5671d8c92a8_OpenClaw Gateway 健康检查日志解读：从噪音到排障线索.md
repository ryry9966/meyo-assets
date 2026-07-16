---
title: OpenClaw Gateway 健康检查日志解读：从噪音到排障线索
feedId: 29314
source: 综合讨论
publishedAt: 2026-07-16
---

很多开发者在 OpenClaw Gateway 接上 K8s 或容器编排后都会遇到同一个困扰：访问日志每隔几秒就刷出 `GET /healthz 200`，夹杂在 Agent 调用、MCP 工具路由的日志里，模糊了真正的请求流水。一开始我也只是简单加上日志过滤，直到某次线上 Gateway 被持续标记为不健康并触发 Pod 重建，排查后发现竟是 /readyz 返回了 503，而根因来自一条 MCP Server 的僵尸连接。从那之后，我重新梳理了 OpenClaw Gateway 健康检查日志的完整读法和排障路径，发现它可以成为很好的可观测性信号，而不只是噪音。

## 1. 理解两种健康端点
OpenClaw Gateway 默认暴露两个健康检查路由：
- `/healthz`：存活探针，仅反映网关进程是否存活，几乎永远返回 200，除非主循环阻塞。
- `/readyz`：就绪探针，反映网关是否准备好接收流量，通常会检查上游依赖（配置的 MCP Server 列表、插件管理器、可能的外部缓存或配置中心）。一旦返回非 200，服务网格就会将 Pod 从端点摘除或重启。

日志里对这两个路径的记录行为取决于 `access_log` 的配置。默认情况下，所有经过 HTTP Handler 的请求（包括 kube-probe）都会被写入 access log，输出 JSON 或文本格式。

## 2. 从日志堆里捞出健康检查
如果你使用文件日志，最快的方式是：
```bash
grep -E '/healthz|/readyz' /var/log/openclaw/gateway.log
```
通过 jq 进一步解析（如果配置为 JSON 日志）：
```bash
grep '/readyz' gateway.log | jq 'select(.status != 200)'
```
当接入 Loki 或 ES 时，可以用如下 LogQL 表达式快速定位异常探活请求：
```
{app="openclaw-gateway"} |~ "/readyz" | status >= 400
```
实际环境里，健康检查请求的典型特点是高频、低延迟，且来源 IP 往往是节点本地或 Sidecar 代理。建议给日志打上 `user_agent: "kube-probe/1.x"` 或自定义请求头，便于区分。

## 3. 看到 503 后怎么办：开启详细健康信息
一次典型的踩坑经历：/readyz 返回 503，但日志里只有一行 `GET /readyz 503`，背后原因完全缺失。这是因为 OpenClaw Gateway 出于安全考虑，默认只在响应体里返回 `not ready`，不暴露内部故障细节。排查时需要在配置中开启详细诊断：
```yaml
gateway:
  health:
    detailed: true
    # 可选：为健康端点启用独立的 listener，避免占用业务线程
    admin_port: 9090
```
重启后，/readyz 在非就绪状态下会输出类似这样的 JSON：
```json
{
  "status": "not_ready",
  "dependencies": {
    "mcp_server:filesystem": "timeout",
    "redis_session": "ok",
    "plugin_registry": "ok"
  }
}
```
同时，日志中也会记录对应组件的健康检查失败事件，如 `health check failed: mcp_server:filesystem context deadline exceeded`。这时你就能精准定位到某个 MCP Server 需要重启或网络策略需要调整。

另一个容易被忽略的问题是**健康检查自身造成的连接池压力**。当 `/readyz` 配置为探测全部依赖，且探活间隔设得过短（如 5 秒），每次请求都会新建到 MCP Server 或 Redis 的连接，这在依赖数量多时会导致 TIME_WAIT 积压，反而拖慢真实请求。解决方法是将 `admin_port` 独立监听，让其连接池与业务端口隔离，同时适当延长 `periodSeconds` 并增大 `timeoutSeconds`。

## 4. 日志级别与清理策略
别让健康检查的 200 日志淹没真正需要关注的错误。可以通过调整 `access_log` 规则过滤掉 `/healthz` 和状态码为 2xx 的 `/readyz`：
```yaml
logging:
  access:
    exclude_paths:
      - "/healthz"
    exclude_status_2xx_on_paths:
      - "/readyz"
```
但务必保留 4xx/5xx 的 /readyz 日志，那是排障的第一现场。此外，将健康检查失败的日志提升到 WARN 级别，并接上 Alertmanager，可以在依赖抖动时提前通知，而不是等服务整体下线才发现。

## 5. 可复用建议
- **所有下游依赖**：确保 MCP Server、插件后端等实现了兼容 OpenClaw 约定的健康端点，并在 /readyz 的依赖列表里显式配置，避免“假健康”。
- **独立管理端口**：在生产环境始终为健康检查和监控指标分配单独的 `admin_port`，防止探活流量抢占业务资源。
- **日志分流**：将健康检查日志（尤其是失败记录）输出到独立的日志文件或索引，便于设置专门的监控和保留策略。
- **模拟探活**：定期用 `curl` 结合 `--connect-timeout` 模拟 kubelet 的行为，确保 `timeoutSeconds` 设置大于最慢的 /readyz 响应时间，避免误杀 Pod。

## 总结
OpenClaw Gateway 的健康检查日志远不止是一堆重复的 200。它就像网关自身运行状况的心电图，遇到就绪探针失败时，只要开启了详细诊断、理清了日志过滤和依赖映射，你能在几分钟内定位到是哪条链路出了问题，而不是无助地看着 Pod 重启。把日志从噪音转化为可行动的排障线索，才是在 Agent 和 MCP 自动化场景下稳定运行 OpenClaw Gateway 的关键一环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/d95e168f69fca850.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/8a7b778a64a9bfa5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/40c2946389691d87.png)

