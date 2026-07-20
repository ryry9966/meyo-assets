---
title: OpenClaw Gateway 健康检查日志：从“报错”到“可观测”的排障路径
feedId: 29842
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在 OpenClaw Gateway 上承载了多个 Agent、MCP 工具和插件路由后，健康检查（Health Check）会变成最容易被“忽视但其实很重要”的信号源。网关需要持续感知后端服务是否可用，日志里就会密集出现 `health check` 相关记录。这些日志既是部署自愈的基础，也是定位服务质量抖动的第一现场，但很多团队把健康检查日志当成噪音，直到某个上游连续 5xx，才发现日志里早有征兆。

健康检查在 OpenClaw Gateway 里有几个典型层级：

- **主动探测**：网关定期向各路由配置的 `health_check_path` 发起请求，记录状态码、延迟、失败次数。
- **被动发现**：根据实际转发请求的结果动态标记后端状态，相关日志出现在 `upstream` 或 `circuit_breaker` 字段。
- **Kubernetes probe**：如果跑在容器环境，readiness/liveness probe 本身也会触发网关自身的健康检查端点，与上游检测混合。

## 典型问题

最常见的情况是：某个 Agent 的后端偶尔被标记为 `unhealthy`，然后几秒后又恢复，网关控制台或监控上出现“抖动”；或者看到一堆 `status: 503` 的健康检查日志，但手动 curl 到后端服务却是正常的。还有一种更隐蔽的场景：上游实际响应 2xx，但延迟超过主动探测的超时，导致日志里大量 `timeout`，而运维只看状态码忽略了耗时。

这时如果直接搜索 `error` 或 `unhealthy`，很容易被海量日志淹没，需要先知道日志里究竟哪些字段能指路。

## 做法与步骤

### 1. 定位一次完整的探测记录

以 OpenClaw Gateway JSON 格式日志为例，一次主动健康检查通常会输出类似：

```json
{
  "level": "debug",
  "component": "healthchecker",
  "route": "mcp-tool-weather",
  "upstream": "10.0.2.15:8080",
  "check_url": "/healthz",
  "status": 200,
  "latency_ms": 42,
  "result": "healthy",
  "timestamp": "2025-03-12T10:23:05.123Z"
}
```

如果结果是 `unhealthy`，会附带 `error_reason`（如 `connection refused`、`timeout`、`status_code: 503`）。先用 `grep` 或日志平台过滤 `healthchecker` 组件，缩小范围。

### 2. 聚类错误原因而非只看状态码

建议按 `error_reason` 或 `status` + `latency_ms` 聚合。例如使用 `jq` 从 JSON 日志流中提取关键信息：

```bash
cat gateway.log | grep healthchecker | jq -r '[.error_reason, .status, .latency_ms] | @tsv' | sort | uniq -c | sort -rn
```

经常看到的问题组合：

- `connection refused` + `latency_ms: 0`：后端根本没在监听，或是滚动更新时旧 Pod 被提前终止。
- `timeout` + `latency_ms: 5000`（等于探测超时）：探针超时设得比后端实际处理时间长，需要调大探测超时或检查后端为何慢。
- `status: 503` + `latency_ms: 10`：后端主动返回 503，可能要查服务内部状态，而不是网关。

### 3. 分清 active 和 passive 检查日志

OpenClaw 的被动健康检查日志埋在转发错误里，字段通常是 `upstream_error` 和 `circuit_state`。如果你看到 `circuit_state: open` 紧接着 healthcheck 报 `healthy`，意思是断路器因错误率过高打开，但主动探测依然返回成功（探测地址可能和真实业务端点不同）。这时候要分别关注两类日志：**健康检查端点只代表“服务进程存活”，不代表业务逻辑正常**。

### 4. 调高日志级别观察瞬时翻转

临时将 gateway 日志级别调到 `debug`（生产谨慎使用），可以看到每次探测的决策过程，比如：

```
healthchecker: route=mcp-tool-weather previous=healthy current=unhealthy consec_failures=3 threshold=3 -> marking unhealthy
```

这对判断阈值设置是否合理很关键。如果 `consec_failures` 刚到 3 就切换，但失败只是因为偶发的 CPU 尖刺，可考虑将阈值调到 5，并配合 `interval` 和 `timeout` 的调整。

## 踩坑点

1. **路径配置重叠**  
   多个路由用了同一个 `/health` 探测路径但指向不同后端？网关可能把健康检查请求误发到错误的上游，导致一整套路由被“连坐”。在配置中尽量让 `health_check_path` 唯一化或与路由明确绑定。

2. **Kubernetes probe 与网关探测相互干扰**  
   如果 readiness probe 指向网关自身的健康端点，而该端点又聚合了所有上游探测结果，就可能出现“上游抖动导致网关 Pod 被重启”的链路。建议网关自身探针只检查进程和基础依赖，不要串联业务上游。

3. **主动探测超时比转发超时更短**  
   默认探测超时常设为 2s，但如果某个 Agent 的真实响应 P99 是 2.5s，就会产生大量误报。务必让主动探测超时 ≥ 后端 P95 延迟，或为重型接口单独配置探测值。

4. **日志级别被全局压制**  
   生产环境常常把 health checker 日志级别设成 `error`，结果只有 `unhealthy` 时才有输出，丢失了连续失败前的缓慢退化信号。建议至少保持 `info` 级别，并利用结构化日志做过滤，而不是靠降级来“省资源”。

## 可复用建议

- **让健康检查自带 trace context**：在 probing 请求头里注入一个固定标识（如 `X-Health-Check: openclaw-gateway`），方便后端日志里快速区分探测流量与业务流量，避免统计污染。
- **监控“健康检查成功率趋势”而不是二值化状态**：例如连续 30 秒内探测成功率 < 80% 就告警，而不是等三次失败才反应。
- **用脚本聚合关键字段**：写一个简单的 CLI 工具或 jq 模板，让团队成员都能一行命令获取健康检查健康度总览，降低排障门槛。
- **将探测端点与真实接口分开**：比如 Agent 提供 `/health` 仅做轻量检查，避免在探测路径里跑完整的推理调用；另外再暴露 `/internal/health` 做深度检查供按需使用。

## 总结

健康检查日志不是“有错误看就行”，它本质上是网关与后端的持续对话记录。把这些日志从告警噪音变成可观测信号，关键在于：区分主动/被动检查、聚类错误原因而非仅看状态码、处理好探测超时与后端时延的关系，并警惕 Kubernetes 探针的连锁效应。下次再看到健康检查频繁翻转，不要急着忽略——你很可能已经抓住了某个即将爆发的耦合故障的尾巴。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/09432f524c6e3280.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/bcd67d9b2d4addd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/cdb18330233678b7.png)

