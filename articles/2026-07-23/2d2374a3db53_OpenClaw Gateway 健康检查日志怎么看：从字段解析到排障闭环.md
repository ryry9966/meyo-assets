---
title: OpenClaw Gateway 健康检查日志怎么看：从字段解析到排障闭环
feedId: 30189
source: 综合讨论
publishedAt: 2026-07-23
---

# OpenClaw Gateway 健康检查日志怎么看：从字段解析到排障闭环

## 背景
在 OpenClaw 体系的 Agent 协作场景里，Gateway 承担着 MCP 服务、插件端点、反向代理与负载入口等角色。为了保证调用链路的可用性，Gateway 默认开启了主动健康检查（active health check），并会按探测间隔输出访问日志与健康状态变更日志。但在日常运维中，这些日志常常被忽略，直到上游 Agent 调用超时才发现某个后端已经挂了一段时间。

健康检查日志并不只是“成功/失败”的二进制信号。它包含探测目标、延迟分布、连续失败次数、异常类型乃至 TLS 握手耗时等信息。读透这些字段，可以帮助你在不登录节点、不抓包的情况下，快速定位是网络抖动、端点逻辑错误还是 MCP Server 过载。

## 问题场景
一个典型的故障表象：Agent 调用 `mcp-tools` 时出现间歇性 502。查看 Gateway 的健康检查页面显示“healthy”，但上游已出现请求卡死。翻开健康检查日志，发现类似输出：

```
[healthcheck] target=mcp-tool-server address=10.2.1.14:8080 outcome=success duration_ms=2 status=200
[healthcheck] target=mcp-tool-server address=10.2.1.14:8080 outcome=timeout duration_ms=3000 status=0
[healthcheck] target=mcp-tool-server address=10.2.1.14:8080 outcome=success duration_ms=5 status=200
```

这一段日志很容易被忽略，但如果持续出现交替的 timeout 和快速恢复，往往是上游服务存在 GC 停顿或工作线程耗尽，单次探测通过掩盖了问题。

另一个常见陷阱：Gateway 自身探活间隔设置过短（例如 1s），而下游 MCP Server 的 `/health` 端点是重量级检查（扫描模型列表、校验 API Key），导致健康检查成为 DDoS 源头，反而引发更多超时。

## 做法与步骤
### 1. 确认日志输出位置与级别
Gateway 默认将健康检查日志写入标准输出，受 `log_level` 控制。至少需要设为 `info` 才能看到检查事件的汇总行；设为 `debug` 则可获得每次探测的完整请求响应体（注意生产环境避免持久开启）。

关键配置片段（YAML 示例）：
```yaml
health_checks:
  - name: mcp-tool-server
    endpoint: /health
    interval: 10s
    timeout: 3s
    unhealthy_threshold: 2
    healthy_threshold: 1
```

### 2. 解析核心字段
每条健康检查日志固定包含以下维度，日志行可能以 JSON 或结构化 key=value 格式呈现。

- **target**：匹配配置中的 `name`，运维侧可直接用作筛选关键字。
- **outcome**：`success` / `timeout` / `connection_refused` / `tls_error` / `http_failure`。`http_failure` 时还会附加 `status` 字段。
- **duration_ms**：完整探测耗时，包含 TCP 握手、TLS、请求往返。若该值接近 `timeout` 设置，说明探测本身已处于临界。
- **address**：实际解析后的实例地址。如果使用了服务发现，可据此判断是全部实例异常还是单点故障。
- **unhealthy_count / healthy_count**：在状态变更日志中出现，而非每次探测。用于追踪阈值累积。
- **error**：出现在非成功探测中，包含具体错误信息，如 `dial tcp 10.2.1.14:8080: i/o timeout`。

### 3. 快速排障命令
假设日志输出到 journald 或容器标准输出：

```bash
# 筛选某个 target 的异常探测
grep "target=mcp-tool-server" /var/log/gateway.log | grep -v "outcome=success"

# 统计最近 5 分钟各 target 的 timeout 比率
awk '/healthcheck/ { ts=$1; split($2,a,"="); target=a[2]; split($3,b,"="); outcome=b[2]; if(outcome=="timeout") cnt[target]++} ...'
```

建议在 Dashboard 侧基于日志搭建实时面板，而非手动 grep。

### 4. 关联指标与告警
健康检查日志是最终兜底，应结合 Prometheus 的 `gateway_health_check_failures_total` 指标设定告警。日志中出现的 `unhealthy_threshold` 触发企业微信/钉钉通知时，切记带上 `address` 与 `error` 字段，避免上线救人时多跳一步查询。

## 踩坑点
1. **健康检查“成功”不代表服务健康**  
   很多 MCP 插件的 `/health` 端点只返回 200 与静态字符串，不检查数据库连接、模型可用性。此时，健康检查日志一路 `success`，但真实请求已经大量 500。解法：让 MCP Server 提供 `/ready` 端点，Gateway 健康检查指向该端点，该端点执行业务级深探。

2. **TLS 证书过期导致 `tls_error` 被淹没**  
   在大量实例轮换时，个别实例证书过期，健康检查偶尔失败，但因阈值未达到而被忽略。日志中会出现间歇性 `tls_error`，应单独监控此 outcome 的出现频率。

3. **探测间隔过短造成雪崩**  
   如前所述，若探测端点本身消耗资源，10s 以下间隔可能在下游服务恢复时立刻将其压垮，形成“健康检查杀死服务”的循环。建议通用探测间隔不低于 5s，且探测端点必须轻量。

4. **IPv6 回退导致 duration_ms 异常**  
   Gateway 可能优先尝试 IPv6 连接，超时后才回退到 IPv4，造成日志中 duration_ms 为 2000ms 左右的 timeout 然后成功。这种情况需要调整内核参数或 Gateway 的解析策略。

## 可复用建议
- **标准化健康端点设计**：区分 `/healthz`（网关自身存活性）与 `/ready`（上游业务就绪性），并在日志 target 字段中使用不同 name，便于区分。
- **日志采样与保留**：正常 success 日志在 debug 以上级别可以采样输出（例如每 100 次输出 1 次），避免海量日志冲击存储；状态变更日志必须全量保留。
- **工具链集成**：将健康检查日志结构化为 Elasticsearch/ClickHouse 字段，构建 Grafana 面板，监控各 target 的 p95 探测延迟、超时率、错误类型分布。
- **主动通知带上上下文**：当 unhealthy 事件触发时，同时采集前后 1 分钟的同类日志快照，发送至运维群，避免“刚刚恢复了，日志没了”的尴尬。

## 总结
OpenClaw Gateway 的健康检查日志是一个容易被低估的“传感器网络”。它不仅能告诉你一个后端是否可达，更能通过 duration_ms、outcome 类型与错误细节，描绘出整个 Mesh 网络的抖动特征。投入半天时间梳理现有日志输出、构建结构化解析和告警规则，可以显著降低 Agent 调用链路的故障发现时间（MTTD），也让凌晨的应急响应更有依据。

不要把健康检查日志当作背景噪声。它是分布式 Agent 系统的脉搏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/3d85a1a3120930fc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/776ce441a80ae23c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/a19f00c0ad28038d.png)

