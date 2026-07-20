---
title: OpenClaw Gateway 健康检查日志怎么看：从字段拆解到排障闭环
feedId: 29859
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景：为什么健康检查日志值得单独看

部署 OpenClaw Gateway 之后，最早暴露问题的往往不是业务接口，而是 /health、/ready 这类健康检查端点。容器调度、负载均衡、发布系统都依赖健康检查来决定流量是否打入实例。如果只关注业务日志，可能等到上线后才发现实例被不断摘流，或者滚动更新时一直失败。

更微妙的是，Gateway 的健康检查不是简单的“进程还活着”——它会聚合上游 Agent Runner、插件状态、配置热加载结果等多个条件。因此，怎么读日志、怎么把日志字段和运行时真实状况对应起来，就成了运维中最基础但也最容易踩坑的一环。

本文基于在生产环境调试 OpenClaw Gateway 健康检查的经验，梳理一套可复用的日志阅读与排障流程。

## 问题：日志在，但结论不清晰

常见现象有：
- 健康检查端点返回 200，但上游 Agent 已不可用，实际流量转发持续失败。
- 日志里记录了健康检查成功/失败，但不知道是哪一项检查没通过。
- 滚动更新时，旧 Pod 已经收到 SIGTERM，但健康检查仍然通过，导致流量迟迟不摘除。
- Aggregated health 状态变化频繁，但缺乏历史对比，难以判断是否属于正常抖动。

这些问题表面是关于 HTTP 状态码的疑惑，本质是对健康检查日志结构不够熟悉，没有把日志当成状态快照来使用。

## 做法：拆解一条完整的健康检查日志

一条典型的 OpenClaw Gateway 健康检查日志（结构化 JSON）大致如下：

```json
{
  "ts": "2025-04-15T07:12:03.421Z",
  "level": "info",
  "msg": "health check completed",
  "component": "health",
  "overall": "degraded",
  "checks": {
    "gateway": "ok",
    "agent_runner": "degraded",
    "plugin_manager": "ok",
    "config_watcher": "ok"
  },
  "details": {
    "agent_runner": "ping timeout: dial tcp 10.42.1.18:9090: i/o timeout",
    "latency_ms": 2940
  }
}
```

### 关键字段含义与用法

- **overall**：聚合后的健康结论。取值通常为 `ok`、`degraded`、`unhealthy`。`degraded` 时 /health 可能仍返回 200，但 /ready 可能返回 503，取决于实施。
- **checks.<subsystem>**：每个子系统的独立状态。常用子系统至少包括：
  - `gateway`：自身 gRPC/HTTP 服务可用性。
  - `agent_runner`：与 Agent 运行时（如 runnerd）的连接与心跳。
  - `plugin_manager`：插件进程是否存活，是否有插件报致命错误。
  - `config_watcher`：热加载模块是否正常监听文件变更，是否出现配置解析失败。
- **details**：当某子系统非 `ok` 时，会附具体失败原因。这里是排障的核心。
- **latency_ms**：整个健康检查的耗时，超过一定阈值（如 3000ms）往往预示上游网络或磁盘问题。

### 如何系统性查看

**第一步：按 overall 状态过滤**
在可观测平台或直接用 jq 筛选：
```bash
cat gateway.log | jq 'select(.component=="health" and .overall!="ok")'
```
这样能快速找到所有非健康时刻。

**第二步：查看 details，定位爆炸半径**
观察具体错误信息。例如 `agent_runner` 出现 `ping timeout`，需进一步查 Agent Runner 自身日志，是否发生 GC 停顿、是否被 OOMKill 前瞬间无响应。

**第三步：将日志与就绪探针配置对照**
确认 /ready 端点是否引用了与日志相同子系统的状态。部分团队自定义 /ready 只检查 `agent_runner` 和 `plugin_manager`，那即便 `config_watcher` 失败也视为就绪，容易造成认知偏差。日志里的 `checks` 是真实依据。

## 踩坑点

1. **误将 /health 和 /ready 作用混淆**
   /health 通常返回总体状态，用于人工诊断和监控看板；/ready 决定流量摘除。生产上可能出现 /health 为 `degraded` 但 /ready 仍为 200，因为团队只检查 `gateway` 子系统。正确做法是阅读启动参数或配置，明确两个端点的检查范围。

2. **日志中的 `latency_ms` 被忽略**
   曾遇到过健康检查日志 `overall: ok`，但 `latency_ms` 稳定在 2500ms 以上。排查后发现 config_watcher 每次健康检查都要重新读取远程配置中心，而网络延迟高。虽然状态 ok，但累积耗时会让就绪探针超时，最终被调度器误杀。务必为 latency_ms 设置告警。

3. **agent_runner `degraded` 的级联效应**
   Agent Runner 不可用并不意味着 Gateway 失去了所有功能，插件可能仍可接受调用。如果不希望因为 Runner 临时故障导致整个 Pod 摘流，可通过配置将 Runner 检查从就绪探针中去除，只在日志中记录，避免级联故障。

4. **日志采样不足**
   健康检查每 10～30 秒打一次，如果日志存储周期短或采样率低，可能一眼看过去全是成功记录，漏掉偶发失败。建议以时间趋势图（例如每分钟 degraded 计数）来代替点查日志。

## 可复用建议

- **建立健康检查日志仪表盘**：以 `overall` 状态为时间序列，按子系统拆分子图。当 degraded 占比突增，直接关联到对应子系统的服务端日志。
- **对 latency_ms 设置 P99 告警**，阈值可取健康检查间隔的 70%。若间隔为 10s，则 latency_ms 超过 7000ms 前就应触发警告。
- **将健康检查日志与事件（OOM、Node NotReady）关联**。在 Loki、Elastic 等系统中建立联动查询，一条 degraded 日志背后常有内存压力或节点故障。
- **灰度更新时，严格比对新旧 Pod 的健康检查日志差异**。如果新 Pod 的 `overall` 频繁在 `ok` 与 `degraded` 间抖动，说明新版与上游组件的兼容性可能存在问题。
- **代码化健康检查配置**，将检查范围、超时时间以配置中心管理，避免每次变更都要改 Deployment 探针参数，减少不一致风险。

## 总结

OpenClaw Gateway 的健康检查日志不是可有可无的“心跳记录”，而是一份实时系统拓扑的状态快照。读它的核心方法是：看 overall 判断宏观健康，看 checks 定位哪个子系统有问题，看 details 找到具体原因，最后结合探针配置决定是否影响流量。把这几层读懂，就能从被动接收告警切换到主动预防故障的状态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/999b370a95b321bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/d8c5705af26406ad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/866616bb12eec356.png)

