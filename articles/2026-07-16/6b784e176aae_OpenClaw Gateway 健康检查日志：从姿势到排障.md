---
title: OpenClaw Gateway 健康检查日志：从姿势到排障
feedId: 29241
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

在生产环境或自动化流水线中，OpenClaw Gateway 常作为 Agent、MCP 插件与外部服务的统一入口，健康检查（health check）几乎是所有可观测性基线的起点。无论是 Kubernetes 的 liveness/readiness 探针、负载均衡器的健康检测，还是 CI/CD 中的部署验证，最终都会落到那一行 `/healthz` 或 `/readyz` 的响应上。

然而实际运维中，只看 HTTP 状态码远远不够。健康检查失败时的日志往往是带出根因的第一现场，但因其触发频率高、输出简洁，容易在日志流中被忽略。本文汇总了一套可复用的查看、解析与排障方法，帮你从日志里把 “200 OK” 背后真正发生的事情挖出来。

## 典型问题：日志能告诉我们什么？

假设你遇到以下场景：

- K8s Pod 反复重启，`describe pod` 只显示 `Liveness probe failed: HTTP probe failed with statuscode: 503`。
- 滚动更新后，新 Pod 一直未 Ready，旧 Pod 未被摘除，流量仍然打到旧实例。
- 自定义 MCP 服务器接入 Gateway 后，`/healthz` 偶发性超时。

这个时候直接去看 Gateway 的标准输出或日志文件，常见的行可能类似：

```log
2025-03-28T09:12:03.442Z  INFO healthcheck: component=plugin_registry status=degraded latency=45ms
2025-03-28T09:12:03.448Z  WARN healthcheck: component=mcp_transport status=unhealthy error="dial tcp 10.42.1.9:8090: i/o timeout"
2025-03-28T09:12:03.449Z  ERROR healthcheck: aggregate_status=fail reason="mcp_transport unhealthy"
```

关键信息其实都在这三行里：服务整体不可用（aggregate_status=fail），直接原因是 `mcp_transport` 组件不健康，根因是到某个 MCP 服务的 TCP 连接超时。如果只看 HTTP 响应码，你只知道 “fail”；日志则帮你定位到了组件和网络层。

## 做法 / 步骤

### 1. 确认你的健康检查端点与日志级别

OpenClaw Gateway 默认暴露两个端点：

- `/healthz`：聚合性检查，适合负载均衡器，仅返回 `200` 或 `503`。
- `/readyz`：面向 Kubernetes Readiness，可能额外包含依赖检查（如数据库、消息队列）。

日志详细度由 `LOG_LEVEL` 环境变量或配置文件中的 `logging.level` 控制。建议至少设为 `INFO`；生产排查期临时调整为 `DEBUG`（需注意日志量）。

### 2. 读懂一条健康检查日志的结构

单次健康检查通常会在短时间内输出多条日志，结构如下：

- **心跳前缀**：`healthcheck:`，便于 `grep`。
- **组件字段**：`component=` 标明被检查的子系统（如 `plugin_registry`、`mcp_transport`、`authn_adapter`、`storage_backend`）。
- **状态标签**：`status=` 取值可能是 `healthy`、`degraded`、`unhealthy`。
- **延迟指标**：`latency=` 表示该组件探测耗时，单位 ms。
- **错误详情**：`error="..."` 仅在非健康时出现，原样抛出底层错误。

聚合行同样包含 `aggregate_status=`，取值 `ok` 或 `fail`，并附 `reason=` 汇总第一个不健康组件的名称。

### 3. 常用查询与过滤

- 只看失败记录：  
  `grep "aggregate_status=fail" gateway.log`
- 统计某个组件瞬时故障频率（排查偶发抖动）：  
  `grep "component=mcp_transport status=unhealthy" gateway.log | awk '{print $1, $2}' | uniq -c`
- 提取所有组件的延迟变化趋势（简易版）：  
  `grep "healthcheck: component=" gateway.log | grep -oP 'latency=\K[0-9]+'`

如果日志量大，建议转发至 Loki、Elasticsearch 等集中日志系统，添加结构化解析规则，便于图表化展示组件级的健康状态趋势。

### 4. 复现与主动排障

当发现 `degraded` 或 `unhealthy` 时，可以手动请求端点并带上 `verbose=true` 参数（如果 Gateway 支持），这会返回组件明细 JSON，和日志内容一致，但更适合脚本化。结合日志中的 `error` 字段：

- `i/o timeout`、`connection refused`：下游服务不监听、防火墙、资源耗尽。
- `plugin initialization error`：插件加载失败，检查插件配置与依赖。
- `storage_backend unhealthy`：数据库或缓存不可达，验证连接字符串与网络策略。

## 踩坑点

1. **日志级别噪音**  
   `DEBUG` 级日志会输出每个组件的探测细节，包括每一次 TCP/tls handshake，一天可能产生数百 MB 日志。仅在临时排障时使用，并尽快恢复为 `INFO`。

2. **误读聚合日志的 reason**  
   `reason=` 只显示第一个检查失败的组件，可能掩盖后续组件的问题。例如 `mcp_transport` 修复后可能暴露 `authn_adapter` 也异常，排障时需继续追踪。

3. **健康检查频率过高导致自伤**  
   某些部署将探针间隔设为 1 秒，而 Gateway 的聚合检查中包含了 3 个远程依赖。高并发请求会让远程依赖的时延抖动更突出，触发偶发性 `unhealthy`。建议探针周期不小于 5 秒，timeout 不小于 3 秒。

4. **没有区分 startup/readiness/liveness 日志**  
   Gateway 可能在 startup 阶段尚未初始化完成，直接返回 503，但此时日志中的组件可能显示为 `uninitialized` 而非 `unhealthy`，不会带 `error=`。需要区分是冷启动还是真正故障。

## 可复用建议

- **标准化日志染色**：在 `fluent-bit` 或 `logstash` 配置中添加解析规则，将 `status` 转换为指标：`healthy=1`，`degraded=0.5`，`unhealthy=0`，形成服务健康评分。
- **主动告警分级**：  
  - `degraded` 持续超过 2 分钟 → 告警到运维群。  
  - `unhealthy` 立即触发 PagerDuty/Alertmanager。
- **在 Helm Chart 中固化** Values 示例，让用户自行定义健康检查的 `verbose` 参数开启条件。
- **与自动化流水线集成**：CI 部署完成后，轮询 `/readyz?verbose=true` 并解析 JSON，确认所有组件 `healthy` 后再继续下一阶段。

## 总结

OpenClaw Gateway 的健康检查日志不止是探针成败的记录，更是一份实时的组件级诊断报告。理解它的结构、掌握基本的过滤和排障方法，可以极大缩短从 “Probe failed” 到定位问题组件的时间。建议将其纳入日常的可观测性体系，而非只在故障时匆匆翻看。处理好这部分日志，你的自愈流程和发布信心都会上一个台阶。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/c909fdef3b077e57.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/0ef5ef3baf654c46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/3ee77f4c6f0333ba.png)

