---
title: OpenClaw Gateway 健康检查日志：从读懂到自动化排障
feedId: 29262
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

OpenClaw Gateway 通常部署在 Agent/MCP 工具链的前端，承担着请求路由、协议转换与后端服务健康判定。它的健康检查模块直接决定了上游调度策略——一旦健康检查出现误判，就会触发错误熔断、请求排队或流量绕过，导致原本可用的下游节点被错误摘除。

很多自动化团队对 Gateway 健康检查只有“过了/没通过”的二元认知，忽略了日志中隐含的服务劣化信号。本文以工程实践为出发点，介绍如何正确开启、解析和利用 OpenClaw Gateway 的健康检查日志，并给出可直接落地的告警脚本。

## 问题：日志就在那里，却读不出信号

健康检查默认日志级别较低，通常只输出 INFO 级别的连通性结果。但在生产环境中，仅靠“UP/DOWN”远远不够。常见的困惑包括：

- 某后端间歇性 503，但健康检查日志一直显示成功，误以为网关无故障；
- 日志中频繁出现 `connection refused`，却因与部署重启期间的短暂抖动混淆而被忽略；
- 不确定一次健康检查失败是否已达到熔断阈值，还是只是瞬时网络波动。

这些场景本质上都是因为缺少对日志结构、检查语义与时序上下文的掌握。

## 做法：三步构建可读的健康检查日志体系

### 1. 开启结构化日志并提升日志级别

默认配置可能只输出单行文本，且 passive health check 的观测结果不会显式记录。建议调整为 JSON 格式，并临时提升相关 logger 级别到 DEBUG。

示例 `gateway.yaml` 片段：

```yaml
logging:
  level: debug
  format: json
  outputs:
    - type: file
      path: /var/log/openclaw-gateway/health.log
      category: healthcheck
health:
  active:
    interval: 5s
    timeout: 2s
    concurrency: 3
  passive:
    observe_window: 30s
    error_threshold: 5
  log_policy:
    include_response_body: false
    mask_headers: ["Authorization"]
```

调整后，每条健康检查的结果将包含 `timestamp`、`target`、`check_type`、`status`、`latency_ms`、`error` 和 `upstream` 等字段。

### 2. 理解关键字段与检查语义

- `check_type: active` 是网关主动发起的心跳探测，`passive` 则是通过对真实请求的响应观测得出的被动判断。两者失败含义不同：主动失败可以立即触发摘除，被动失败仅在滑动窗口内达到阈值后才标记不可用。
- `status: UP/DOWN` 是最终判定，但必须结合 `latency_ms` 和 `error` 分析。例如连接超时与连接拒绝对应的 `error` 分别是 `dial tcp ... i/o timeout` 和 `connection refused`，前者可能是下游负载过高，后者往往是进程未启动。
- `consecutive_failures` 与 `threshold` 字段（若配置）能直接展示当前失败计数，避免你手工去 grep 计数。

### 3. 用 jq 做快速统计

假设日志文件为逐行 JSON，可按下游分组统计健康检查失败率及延迟分位：

```bash
cat health.log | jq -r '
  select(.status == "DOWN" or .error != null) |
  "\(.target) \(.latency_ms) \(.error)"
' | awk '{target=$1; lat=$2; err=substr($0, index($0,$3)); count[target]++; sum[target]+=lat; if(err!="") fail[target]++}
   END{for(t in count) print t, "avg_lat:", sum[t]/count[t], "fail:", fail[t]}'
```

更严谨的做法是将日志接入 OpenSearch/Loki，利用可视化面板跟踪延迟趋势与失败率变化。

## 踩坑点

### 误读“TCP 通”为“服务正常”

部分团队将健康检查降级为简单的 TCP 端口探测，日志显示 `status: UP` 就认为后端健康。但 TCP 握手成功只能证明操作系统网络栈正常，服务进程可能卡死、阻塞或返回 5xx。务必使用 HTTP/TLS 应用层检查，并在日志中记录响应状态码和 body 摘要（脱敏后）。

### 忽略 passive 检查的异步特性

被动健康检查的日志打印时机可能晚于真实请求，因为它是基于统计窗口的后置分析。当在日志中看到被动 `DOWN` 时，对应的请求可能已经返回给客户端一个 5xx 或超时。排查时不要只看健康检查日志，需关联 `access log` 中同一时间窗口的异常状态码，才能还原故障发生点。

### 告警阈值照搬文档建议

文档推荐的阈值（如连续 3 次失败）在多租户环境或响应时间抖动的后端可能导致频繁误报。应该根据下游服务的 P99 延迟分布和容忍度设定：例如 10 秒内失败次数超过窗口内请求量的 20%，并结合延迟突增。

## 可复用建议

- **为健康检查日志建立独立保留策略**：健康检查日志高频写入，若与业务日志混存会快速膨胀。建议单独文件，保留 3-7 天，便于回溯近期抖动。
- **编写轻量拨测脚本**：在 CI/CD 或本地诊断时，用 `curl` 结合 Gateway 的管理端口 API 拉取当前健康状态，与日志交叉验证：

```bash
curl -s http://openclaw-gateway:9090/api/v1/upstreams | jq '.[] | {endpoint: .target, healthy: .healthy, failures: .consecutive_failures}'
```

- **基于日志的自动化告警规则**（Prometheus Alertmanager 示例思路）：
  - 近 2 分钟 active check 失败次数 > 2 且为同一 endpoint
  - passive check 在 1 分钟内连续标记 3 个 endpoint 为 DOWN
  - P99 延迟超过下游 SLA 的 150%

将这些规则同步给值班手册，避免仅凭表面日志臆断。

## 总结

OpenClaw Gateway 的健康检查日志是连接监控与调试的桥梁。从开启结构化日志、区分检查语义，到提取延迟/错误特征并建立自动化告警，每一步都能让团队更早感知依赖劣化。不要在出现全局 502 后才去 grep `timeout`，而是在 P99 延迟爬上台阶的第一刻就收到通知——这才是健康检查日志真正的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/343d433bc97ff2b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/53be486f9c499e7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/4a9376312d336280.png)

