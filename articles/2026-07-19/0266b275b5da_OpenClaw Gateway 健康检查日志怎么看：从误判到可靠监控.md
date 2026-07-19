---
title: OpenClaw Gateway 健康检查日志怎么看：从误判到可靠监控
feedId: 29619
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

OpenClaw Gateway 作为 Agent 与插件体系的核心流量入口，通过内置健康检查端点 `/health` 向上游负载均衡、Kubernetes liveness/readiness probe 以及运维面板暴露服务状态。健康检查日志是日常排障最先翻看的信号源，但实践中常被过度简化——只要看到 `200 OK` 就觉得一切正常。真实场景更微妙：依赖的 MCP 服务器延迟升高、某类插件未就绪、线程池积压，都可能让健康检查仍返回 200，但服务实际已处于半死不活的状态。

这篇文章面向已经在用 OpenClaw 做 Agent 编排、挂接自定义插件、对接外部 MCP 服务器的工程同学，梳理如何读懂 Gateway 的健康检查日志，避开几个常见的误判陷阱，并建立一套可复用的监控策略。

## 问题

默认情况下，OpenClaw Gateway 在 `INFO` 级别会周期性打印类似下面的健康检查日志：

```
2025-03-12T10:23:17.432Z  INFO 1 --- [nio-8000-exec-3] o.o.gateway.health.HealthIndicator       : Health check from 10.1.2.3 - total: UP, components: {gateway=UP, dispatch=UP, pluginRegistry=UP, mcpConnectors=UP}
```

运维同学通常会抓住那个 `UP` 就开始写告警规则：`health check != 200 → alert`。但很快你会发现凌晨三点收到一条“服务 DOWN”的告警，一看日志，其实是某个不关键的 MCP 连接器暂时超时，Gateway 仍能正常处理主要流量。或者相反，某个核心插件加载失败，但健康检查仍然 `UP`，因为框架认为插件隔离在沙箱中，不影响整体存活。

问题的根源在于，OpenClaw Gateway 的健康模型是一个**组合健康状态**，各模块的权重和影响范围并不相同，而默认的 `/health` 端点只给出整体结论。如果把组合结果粗暴等于服务可用性，就会出现大量误报和漏报。

## 做法与步骤

### 1. 打开详细健康检查端点

OpenClaw Gateway 支持详细模式，在配置中开启 `openclaw.health.detail=true` 后，再访问 `/health/detail` 可以得到每个子模块的独立状态和额外信息。对应的日志行会包含组件级别的异常原因：

```
WARN  Health check: gateway=UP, dispatch=DOWN (reason: dispatch thread pool exhausted, active=64, max=64), pluginRegistry=UP, mcpConnectors=DOWN (reason: mcp://external-llm timeout after 3000ms)
```

这一步是排障必做的。在日志采集管道中，确保能够结构化解析出组件名、状态、失败原因，后续告警才能精确。

### 2. 定义三种健康等级

不要用一个布尔值判断健康。将组件状态映射为三个等级：

- **CRITICAL**：缺少则服务不可用的模块，例如 dispatch（调度器）、gateway 自身、核心存储连接。任一 DOWN 则整体应视为不可用。
- **WARNING**：影响部分功能但不阻断主链路，例如非缓存的 MCP 连接器降级、某个低频插件加载失败。对应返回状态仍可为 200，但需发出低优先级告警。
- **INFO**：纯旁路模块，如审计日志写入失败、后台统计任务报错，不应改变复合健康状态。

在 Gateway 的 health indicator 配置中，可以通过 `openclaw.health.groups` 将组件标记到不同分组，让 `/health` 只聚合 CRITICAL 组件，而 `/health/warning` 和 `/health/detail` 各司其职。

### 3. 解析日志中的时间窗口信息

OpenClaw 的健康检查不是每次请求都从头探测所有依赖，而是基于滑动窗口缓存。日志中你会看到 `lastProbeAt` 和 `probeInterval` 字段（如果配置了输出）。例如：

```
mcpConnectors=DOWN (lastProbeAt=2025-03-12T10:23:14Z, probeInterval=10s, reason: timeout)
```

这意味着你在 `10:23:17` 看到的 DOWN 状态，其实是 3 秒前那次探测的结果。如果上游 MCP 服务器刚好在探测结束后恢复，而下一次探测还没触发，此时健康检查仍会认为故障存续。理解这个延时，可以避免在误报时白白重启 Pod。

### 4. 联调自建健康检查端点

如果你的 Agent 插入了自定义逻辑（例如必调的外部 API、本地模型预热），建议利用 OpenClaw 的 SPI 扩展实现一个 `HealthContributor`，把自定义健康逻辑注入到组合模型中，这样日志里就会出现 `custom=UP/DOWN`，而不用另起一套监控。确保关键业务前提直接体现在 Gateway 的健康判断中，避免“网关活着但业务走不通”的困局。

## 踩坑点

**坑1：把 `/health` 同时用作 liveness 和 readiness**  
Kubernetes 下常见的错误是 liveness probe 也指向 `/health`，导致当某个 WARNING 级依赖故障时 kubelet 误杀 Pod，重启流量雪崩。正确做法：liveness 指向一个仅包含 CRITICAL 组件的极简端点，readiness 可以使用 `/health/detail` 并配合 `failureThreshold` 容忍短暂抖动。

**坑2：HTTP 状态码与业务语义脱钩**  
在部分配置下，即使所有 CRITICAL 组件 DOWN，Gateway 仍可能因为异常处理框架的兜底而返回 200，仅 body 中 `status=DOWN`。如果负载均衡器只看状态码，就会把已死的节点留在转发列表里。务必让健康检查响应在故障时真正返回 503，可通过 `openclaw.health.http-down-status=503` 显式设置。

**坑3：日志采样导致关键信息丢失**  
高并发场景下，健康检查日志可能被采样或合并，导致偶尔的抖动日志被吞掉。建议对 `gateway.health` 包设置独立的日志级别为 `WARN`，并将这类日志路由到单独的索引，避免淹没在业务日志中。

## 可复用建议

1. **建立分层告警**：基于健康等级，CRITICAL DOWN → P1 紧急告警；WARNING DOWN 持续 2 个周期 → P3 通知；INFO 故障仅记录仪表盘。
2. **结构化日志模板**：统一在 Gateway 侧以 JSON 格式输出健康检查行，确保字段包含 `component`、`status`、`reason`、`latencyMs`，方便 ELK/Loki 解析。
3. **探针超时放大**：上游负载均衡对健康检查的超时设置应大于内部探测间隔的 2 倍，避免探测本身积压导致误报连锁反应。
4. **定期演练故障注入**：在预发环境手动断开某个 MCP 连接器，观察日志链路是否符合预期，检验告警是否按等级触达，而不是等到线上出事故才验证健康检查模型。

## 总结

OpenClaw Gateway 的健康检查日志不止是“活着还是死了”的二值信号，它暴露了调度器负载、插件注册状态、MCP 连接器可用性和自定义业务探针的实时快照。正确的读法是：先切换到详细视图，为组件分级，理解探测缓存窗口，再据此切割 liveness/readiness 端点，并把日志结构化和告警分层落地。这一套做完，凌晨的三点告警才能真正帮你定位问题，而不是把你叫起来看一个假阳性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/0c3aeef4e202e5ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/bb28fd9a0af60403.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/ac5ae0744e84a5d8.png)

