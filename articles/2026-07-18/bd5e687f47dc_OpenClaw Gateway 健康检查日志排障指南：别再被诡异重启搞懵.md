---
title: OpenClaw Gateway 健康检查日志排障指南：别再被诡异重启搞懵
feedId: 29455
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

在 OpenClaw 体系里，Gateway 负责承接所有 Agent、MCP 插件的入口流量，起路由、鉴权、限流、协议转换等作用。这类组件一旦部署到生产，几乎都会配上 Kubernetes 的 Liveness / Readiness 探针，指向类似 `/healthz`、`/readyz` 的路径。于是，每 10 秒左右就会有几条健康检查请求打在 Gateway 上。

平时这些请求悄无声息，可一旦 Pod 开始频繁重启，或者 READY 状态反复横跳，同事的第一反应往往是：“是不是健康检查报错了？日志里能看到啥？” 如果你翻出来的日志只有满屏的 `GET /healthz 200`，既没有耗时也没有任何上下文，那基本等于没看。本文就围绕这一场景，梳理如何真正读懂 OpenClaw Gateway 的健康检查日志，从噪声里抽出有用信号。

## 问题：有日志但找不到原因

常见的“无效日志”大概长这样：

```
192.168.1.1 - - [15/May/2025:08:32:10 +0000] "GET /healthz HTTP/1.1" 200 2
```

单看这条，你只能知道探针成功了一次。但如果 Pod 刚刚被 kubelet 杀掉重启，这条日志既看不出为什么之前的 `/readyz` 失败，也看不出是不是依赖的 Redis / MCP 后端超时导致了就绪检查返回 503。

更糟糕的是，有的 Gateway 实例会把健康检查日志按 DEBUG 级别输出，生产环境为了性能关掉了，出问题时又不得不临时打开，同时还要承担日志量暴增、磁盘打满的风险。

所以，有效看日志的第一步，不是闷头 grep，而是先确认你的 Gateway 到底在健康检查端写了什么、写到了哪里。

## 做法：三步看懂健康检查日志

### 1. 确认日志载体和结构

OpenClaw Gateway 内部通常使用 `slog` 或类似的日志库，健康检查请求会以 Apache Common Log 格式或结构化 JSON 输出，取决于配置。推荐使用结构化日志，后续方便用 `jq` 过滤。

在 `gateway.yaml` 或环境变量中，确认以下设置：

- 日志级别：至少 INFO，线上建议 INFO，不要用 DEBUG 打满。
- 访问日志开关：`access_log: true`，并配置 `health_check_path` 白名单，使其仍然输出但可以单独处理。
- 关键字段：必须包含 `status_code`、`latency_ms`、`path`、`method`。

典型的结构化日志条目看起来是这样：

```json
{
  "time": "2025-05-15T08:32:10.123Z",
  "level": "INFO",
  "msg": "request completed",
  "method": "GET",
  "path": "/healthz",
  "status": 200,
  "latency_ms": 2.4
}
```

### 2. 用组合 grep / jq 快速定位异常

假设 Pod 名包含 `openclaw-gateway`，日志存储在 stdout，可以用 `kubectl logs` 配合 `jq` 聚焦健康检查端点的问题：

```bash
kubectl logs deployment/openclaw-gateway --tail=5000 | \
  jq 'select(.path == "/healthz" or .path == "/readyz")' | \
  jq -s 'group_by(.status) | map({status: .[0].status, count: length, max_latency: (max_by(.latency_ms).latency_ms)})'
```

这个管道会按状态码分组统计，还能看到每个状态码下的最大延迟。如果某段时间 `/readyz` 状态码出现 503，立刻就能抓到；如果 `latency_ms` 超过你探针配置的 `timeoutSeconds`（比如 3 秒），也一目了然。

非结构化日志的话，只能用 `awk` 统计，但前提是日志格式规整，否则路径、状态码、耗时提取会很痛苦。强烈建议在投产前就用 JSON 格式。

### 3. 关联上下游依赖

大多数时候健康检查失败不是因为 Gateway 自己挂了，而是它的就绪探针里串了外部依赖：比如连一次 Redis、Ping 一下 MCP Server 的 `/health`。这类逻辑一旦踩坑，会导致一个依赖抖动就拉下一批 Pod。

排查时，在日志中找到 `/readyz` 返回 503 的时间点，然后往前找这个时间窗口内所有依赖调用的错误日志。例如：

```bash
kubectl logs ... | jq 'select(.time >= "2025-05-15T08:30:00Z" and .time <= "2025-05-15T08:33:00Z")' | \
  jq 'select(.level == "ERROR" or (.msg | test("redis|mcp|timeout"; "i")))'
```

如果看到 Redis 连接超时或 MCP 后端返回 502 紧接着多个 Pod 的 Readiness 失败，基本可以断定是由于探针过“重”引发的连锁反应。

## 踩坑点

- **探针逻辑过重**：在 `/readyz` 里做全量依赖健康扫描，导致 10s 一次的探针把下游打死，或者一个慢查询就触发探针超时。生产环境务必保持探针轻量，只验证自身状态和关键连接是否存活，不做深度业务检查。

- **日志级别误伤**：为了压住“无意义”的健康检查日志，有人会在路由层直接丢弃 `/healthz` 请求的日志。结果就是 Pod 重启后，完全没有任何探针记录，连什么时候失败的都不知道。正确做法是允许日志输出，但可以通过采集侧过滤或日志保留策略处理。

- **初始延迟设置不当**：`initialDelaySeconds` 配得太短，Gateway 还没完全初始化好就被探针打成不健康；或者配得太长，导致真正的启动失败迟迟没被发现。这个值要根据实际的启动耗时（可以从日志时间戳差值得出）来设定，并留出 5~10 秒缓冲。

- **Liveness 和 Readiness 混淆**：Liveness 失败会被重启，Readiness 失败只会摘流。如果让同一个端点充当两者，偶发的 Readiness 失败就会引发 Pod 重启风暴。务必区分路径：`/healthz` 做存活检查（简单 local 状态），`/readyz` 做就绪检查（允许依赖），而且两者日志要区分路径字段便于检索。

## 可复用建议

1. **日志白盒化**：确保每个健康检查请求的日志至少包含 `path, status, latency_ms`，并设为 INFO 级别。可以在 Gateway 内部对健康检查路径使用单独的日志记录器，默认输出，但通过日志采集区分索引或主题。

2. **探针模拟脚本**：写一个简单的 `health-probe-sim.sh`，在容器内用 curl 分别请求 `/healthz` 和 `/readyz`，输出状态码和响应时间，配合 `sleep` 模拟探针频率。在变更依赖或上线前跑十分钟，能提前发现很多问题。

3. **告警规则**：在日志监控（如 Loki+Alertmanager）中加上两条规则：  
   - 过去 5 分钟内 `/readyz` 非 200 次数 > 2 → 告警  
   - `/readyz` 的 P95 延迟 > 2s 且持续 3 分钟 → 告警  
   这样不用等到 Pod 重启就能知道健康检查在恶化。

4. **依赖隔离**：如果确实需要在 `/readyz` 里检查 Redis，加一个独立于业务连接的 check-only 连接，设置 500ms 超时，失败直接返回 503，避免被业务长连接阻塞。

## 总结

OpenClaw Gateway 的健康检查日志不该只是“200 OK”的滚动屏，它承载着 Pod 生命周期和流量摘除的关键决策信息。当你再遇到无头重启时，别急着改 `failureThreshold` 或 `periodSeconds`，先用结构化日志把 `/healthz` 和 `/readyz` 的状态码分布、延迟变化拉出来，再顺藤摸瓜看依赖错误。让日志真正服务于可观察性，而不是用来凑数。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/dd1ee9bd6fff5dc0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/3a483d0dde3a8cd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/872fb6723cade925.png)

