---
title: OpenClaw Gateway 健康检查日志解读指北
feedId: 30216
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在生产环境，OpenClaw Gateway 作为 Agent 编排与插件调用的统一入口，会通过健康检查持续探测后端服务的可用性。健康检查不只看服务是否“活着”，还影响流量路由、熔断决策和自动摘除。而这一切的线索都藏在健康检查日志里。

很多团队只在服务挂掉时才想起看日志，却因为不熟悉字段含义、缺失上下文配置，导致排障时间拉长。本文从一个真实排查过程出发，梳理如何系统化地读懂 Gateway 的健康检查日志，形成可复用的观测习惯。

## 一个典型问题

某次上线后，发现一部分 Agent 节点负载不均——有些节点根本没收到流量。检查 Endpoints 列表，对应的 Pod IP 明明在。打开 Gateway 日志，看到大量 `Health check failed` 记录，但 curl 单独访问这些后端是正常的。为什么 Gateway 认为它不健康？

类似这种“能通但被摘除”的场景，背后往往是对健康检查机制理解不足，而日志是离真相最近的入口。

## 日志长什么样

OpenClaw Gateway 默认使用结构化 JSON 日志，健康检查相关记录通常包含以下关键字段（以 v2.3 版本为例，实际字段视插件/协议略有差异）：

```json
{
  "ts": "2025-03-17T10:23:05.231Z",
  "level": "info",
  "msg": "health check result",
  "upstream": "agent-executor-2.namespace:8080",
  "check_type": "http",
  "status": "unhealthy",
  "status_code": 0,
  "error": "context deadline exceeded",
  "latency_ms": 2001,
  "consecutive_failures": 3,
  "total_checks": 15
}
```

### 核心字段解读

- **check_type**：目前支持的探测协议有 `http`、`grpc`、`tcp`，插件化扩展后也会出现自定义协议。不同协议在失败时的错误信息差异很大。
- **status**：`healthy` / `unhealthy`，是路由决策的直接依据。但要注意，Gateway 内部可能有“半开”或“退化”状态，这些不会直接体现在这个字段上，需要看 `consecutive_failures` 累加值。
- **status_code**：仅对 HTTP 探测有效。如果看到 0，通常是连接未建立或超时，此时应当结合 `error` 字段。
- **error**：原始的 Golang error 信息，比如 `dial tcp 10.0.1.5:8080: connect: connection refused` 或 `context deadline exceeded`。这是排障第一线索。
- **latency_ms**：单次探测延迟，配合 `timeout` 配置可以判断是服务慢还是网络慢。
- **consecutive_failures / total_checks**：这两个计数器帮助确认是偶发抖动还是持续故障。当 `consecutive_failures` 达到阈值（默认 3）后，该上游会被标记为不健康。

## 常见异常及对应排查动作

### 1. `connection refused`
服务未监听对应端口或 Pod 正在终止。先确认容器内进程是否真的启动。如果发生在滚动更新期间，检查 `terminationGracePeriodSeconds` 和 preStop hook 是否给 Gateway 留下了足够的摘除时间。日志中一连串 `connection refused` 后突然消失，往往是 K8s 已经把旧 Pod 干掉但 Gateway 还在探测。

### 2. `context deadline exceeded`
超时，但超时发生的位置需要区分：是 Gateway 发出的探测请求到达了服务但没及时回复，还是连建立连接都没完成？如果是后者，通常 `latency_ms` 会略小于 timeout，且没有 `status_code`。这时优先检查网络策略（NetworkPolicy）、CNI 延迟或对端半连接队列溢出。

### 3. `status_code` 返回 503 或非预期状态码
如果后端在不符合预期的情况下返回 503（如 Java 应用的 thread pool 满），Gateway 会判定为 unhealthy。此时不仅要看 Gateway 日志，还需要联动后端日志和 metrics，找到 503 的根因。可以在健康检查接口中增加更细粒度的内部检查（DB 连接、Redis 连通性），在返回 503 前提前暴露问题。

### 4. 偶发 `unhealthy` 后立刻恢复
日志会出现 `status` 在短时间内反复翻转。如果 `latency_ms` 接近 timeout 阈值，很可能是 GC 停顿或瞬时负载抖动。可适当调大 timeout 和失败阈值，但更推荐的做法是优化后端服务的 tail latency，而不是靠放宽检查标准来掩盖问题。

## 踩坑点

### 坑1：忽视了就绪探针与 Gateway 健康检查的协同
K8s 的 readiness probe 和 Gateway 的健康检查是两个独立系统。可能出现 K8s 认为 Pod ready（端口通），但 Gateway 因 HTTP 返回码不符合要求而摘除的现象。正确的做法是让两者的检查逻辑对齐：使用相同的接口、相同的成功标准。

### 坑2：未启用调试级别日志
生产环境习惯只开 `info`，但健康检查失败的根本原因有时需要 `debug` 级别的请求/响应体 dump 才能定位（比如返回 body 中的自定义错误码）。建议在故障期间临时打开 `debug` 采样，或利用动态日志级别能力，用完及时关掉，避免日志爆炸。

### 坑3：把主动探测和被动健康混为一谈
OpenClaw Gateway 支持主动探测（定期发起）和被动探测（通过真实请求的异常率推断）。日志里 `check_type` 只有主动探测的结果，被动健康变化的线索会散落在请求错误日志中。如果只看主动探测日志而忽略请求流日志，可能漏掉“服务半死”的情况。

## 可复用建议

1. **建立健康检查日志的聚合视图**  
   将 Gateway 日志接入现有日志平台，按 `upstream` 聚合，计算 unhealthy 比例、P99 延迟。设置告警规则：单实例连续 2 次失败或 5 分钟内 unhealthy 比例 > 10% 就触发通知。

2. **携带 requestId 进行全链路关联**  
   在健康探测请求中加入唯一 header，将这个 ID 打印在 Gateway 日志和后端日志里。这样当探测失败时，可以一次性拉出整条链路上的日志，不用靠时间戳猜。

3. **画一张“决策树”**  
   团队内部整理一张根据错误类型和 metrics 排障的决策图（见下文图片提示）。让新手值班时能按图索骥，缩短 MTTR。

4. **定期演练摘除与恢复**  
   人工注入故障，观察日志变化是否与预期一致。这能验证告警规则和日志配置的有效性，也能让团队对健康检查的工作方式形成肌肉记忆。

## 总结

健康检查日志本质上是分布式系统“心跳”的记录。它的价值不在于记录正常，而在于如何快速从异常信号中定位到真正病灶。建议把本文提到的字段、踩坑点和排查动作编入 team runbook，配合决策图和聚合监控，就能把健康检查日志从“出问题才看”变成“日常可观测”的一部分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/ba0c5e0cfbef5c90.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/b535b8767dc945ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/65264586f5fa2c64.png)

