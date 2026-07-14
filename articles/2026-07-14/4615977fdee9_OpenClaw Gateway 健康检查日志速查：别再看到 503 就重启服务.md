---
title: OpenClaw Gateway 健康检查日志速查：别再看到 503 就重启服务
feedId: 28991
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景

在 OpenClaw 的 agent-plugin-mcp 链条中，Gateway 充当了所有外部工具的流量入口与健康守门人。无论是自建的 MCP Server、第三方 API 封装插件，还是内部 RPC 服务，只要注册到 Gateway，都会被周期性的健康检查（health check）持续探测。一旦某后端被标记为 `unhealthy`，Gateway 会自动将流量摘除，agent 侧看到的则是工具不可用、任务中断。

不少同学看到 Gateway 日志中出现连续 503 或健康检查失败，第一反应是“服务挂了，重启”。但大量 Case 其实只是配置不当、探活路径不存在，或者网络环境差异导致误判。真正需要重启服务的场景反而只占一小部分。因此，学会读懂健康检查日志，比盲目重启更重要。

下面用一个真实工程环境中复现的排查过程，把日志字段含义、排障路径和可复用实践串起来。

## 日志长什么样，先看结构

OpenClaw Gateway 默认会为每个 route（即每个注册的后端工具）输出健康检查日志，格式大致如下（实际字段名因版本略有差异）：

```
level=info ts=2025-03-17T02:15:10Z msg="health check result" 
route=mcp/github-issues 
endpoint=http://10.0.1.23:8090/health 
status=503 
latency_ms=3012 
error="context deadline exceeded" 
consecutive_failures=3
```

关键字段解读：
- **route**：在 Gateway 中注册的唯一标识，对应哪个工具或插件。
- **endpoint**：健康检查实际请求的 URL。这个路径是否正确，是最先要确认的。
- **status**：HTTP 状态码。如果不是 200–399，就会被记为一次失败。
- **latency_ms**：响应耗时。若接近或超过配置的 `timeout`，基本就是超时类失败。
- **error**：Go 标准库返回的错误信息，如连接拒绝、TLS 握手失败、上下文超时等。
- **consecutive_failures**：连续失败计数。到达 `unhealthy_threshold` 后，该 route 会被标记为不健康。

**主动检查 vs 被动检查**  
Gateway 默认同时启用主动检查（定时往 /health 发请求）和被动检查（真实调用失败也计入健康状态）。日志中 `msg="health check result"` 是主动检查；`msg="passive health check update"` 则是流量侧触发的。排障时要分清来源：后者可能在偶发超时后自行恢复，不代表服务不可用。

## 排障实操：五个步骤定位问题

### 1. 确认健康检查端点是否真正可达
登录到 Gateway 所在容器或机器，直接用 `curl` 模拟探活请求：
```bash
curl -v --max-time 5 http://10.0.1.23:8090/health
```
很多 MCP 工具没有实现 `/health` 端点，比如某些 streamable HTTP 服务只暴露 `/sse` 或 `/message`。如果 404 或 405，直接在 Gateway 配置中将健康检查路径改到实际可达的轻量接口（如 `/ping` 或 HEAD `/`）。

**踩坑点**：在 k8s 中，Gateway 多跑在集群网络，而 MCP 服务可能仅监听 `127.0.0.1`。此时 `curl` 成功是因为在 Pod 内执行，但 Gateway 可能使用 Pod IP 访问，需确认服务监听 `0.0.0.0`。

### 2. 检查 timeout 与 unhealthy threshold 是否匹配实际延迟
如果日志中 `latency_ms` 经常在 2800–3100ms，而 timeout 设为 3000ms，就会出现偶发性超时。尤其 GPU 推理类工具的首请求耗时可能较长。建议将主动检查 timeout 调整为 P99 延迟的 1.5 倍，`unhealthy_threshold` 从默认 3 改为 5 或更高，避免因网络抖动频繁摘除节点。

### 3. 区分连接失败与 TLS 错误
`error="connection refused"` 多半是服务进程没起来或端口错误；`error="x509: certificate signed by unknown authority"` 则是 mTLS 或自签证书未在 Gateway 侧正确信任。解决方法是添加 `insecure_skip_verify: true`（仅限内网）或挂载正确的 CA 证书。

### 4. 观察被动检查日志辅助判断
主动检查可能被防火墙或服务网格拦截，但真实流量可能正常。如果看到 `passive health check update` 每次都将同一 route 标记为不健康，而主动检查却 200，很可能是客户端请求参数导致的 500，这类问题要回到工具实现层面排查，而不是网关。

### 5. 动态调整日志级别
默认 `info` 级别可能看不到健康检查的具体请求细节。临时将日志级别提到 `debug`（重启 Gateway 或热加载配置），能看到完整请求头、响应体开头，更易定位如认证头缺失、返回 JSON 未被识别等问题。问题解决后记得切回 `info`，避免日志洪流。

## 可复用建议

- **统一健康检查约定**：要求所有注册到 Gateway 的工具都提供 `GET /healthz` 且返回 200 + `{"status":"ok"}`。并在工具开发模板中固化，避免临时猜测。
- **结构化日志接入**：将 Gateway 的健康检查日志输出到 Loki/Elasticsearch，通过 `route` 和 `consecutive_failures` 字段建立告警，例如“任一 route 连续失败 3 次且持续 2 分钟”通知。
- **利用 Gateway 的 Metrics**：`openclaw_gateway_health_check_failures_total` 指标能反映趋势，配合 Grafana 面板，可快速区分是全局网络异常还是个别端点故障。
- **预置恢复脚本**：对于已知的非致命错误（如 DNS 解析延迟），可配置 Gateway 自动重试；而不是手工重启。合理使用断路器半开状态让服务有机会自愈。

## 总结

健康检查日志不是“出事才看”的报警器，而是 Gateway 对后端状态持续建模的窗口。理解透几个关键字段的含义，结合端到端的可达性验证和阈值调优，能让大部分“假死”故障在 5 分钟内定位，而不是花半小时重启一圈服务后发现依旧报错。下次再看到连续 503，不妨先打开日志，沿着 `route -> endpoint -> status/error -> consecutive_failures` 的路径走一遍，答案往往比想象中简单。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/f6bebdb4450c2bd5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/28e96fa0d1adbc03.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/d1f734133397b4a5.png)

