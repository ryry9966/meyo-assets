---
title: OpenClaw Gateway 健康检查日志怎么看：从一团乱到可诊断
feedId: 29253
source: 综合讨论
publishedAt: 2026-07-16
---

# OpenClaw Gateway 健康检查日志怎么看：从一团乱到可诊断

## 背景：被忽略的“心跳”日志

在 OpenClaw Agent / MCP 插件的自动化部署里，Gateway 常常是请求的第一跳，也是健康检查最密集的组件。无论是 Kubernetes 的 liveness / readiness probe，还是上游负载均衡器的主动探测，都会高频调用 `/healthz` 或类似端点。每次调用都会写一行日志，默认情况下这些日志极其相似，很容易被当成噪声直接忽略。

真正遇到问题时——比如 Gateway 频繁重启、Pod 状态在 Ready 和 NotReady 之间抖动、下游 Agent 连接意外断开——健康检查日志往往是第一手线索。但多数人的第一反应是去翻 error log，而健康检查日志通常记在 access log 或专门的探活日志里，被淹没在大量 200 OK 之中，很难定位。

本文整理了一套在 OpenClaw Gateway 上可复用的健康检查日志分析方法，不依赖特定可观测性平台，只用命令行和网关自带配置就能做到。

## 问题定义：什么症状该去看健康检查日志？

以下场景值得优先检查健康检查日志：

- Gateway Pod 频繁重启，但业务错误日志中无明显报错
- K8s 集群中 readiness probe 间歇性失败，导致 Service 端点被摘除
- 上游负载均衡器将 Gateway 节点标记为 unhealthy，但节点自身 CPU/内存正常
- 健康检查端点返回 200，但实际转发到 Agent 的请求大量超时（“假健康”）

这些问题的共同点是：健康检查表面“绿”了，但行为与预期严重不符。而日志能回答“检查到底做了什么、用了多久、依赖了什么”。

## 做法与步骤

### 1. 先确认健康检查日志写在哪里

OpenClaw Gateway 的日志输出通常受 `log.output` 和 `log.level` 控制。默认情况下，健康检查相关的日志会落在 `access` 或 `probe` 分类器下。可以在 `gateway.yaml` 中确认：

```yaml
log:
  output: stdout
  level: info
  access:
    enabled: true
    format: json
  probe:
    enabled: true        # 独立开关，建议开启
    destination: file    # 可选 stdout / file
    path: /var/log/openclaw/probe.log
```

如果没有显式的 `probe` 配置，健康检查请求会作为普通请求记入 access log，需要靠 `path` 字段过滤。

### 2. 用最简单的 grep 提取有效信号

假设日志格式为结构化 JSON，每条记录包含 `path`、`status`、`latency_ms`、`upstream` 等字段。可以先缩小范围：

```bash
# 只看健康检查路径
grep '"path":"/healthz"' /var/log/openclaw/probe.log | tail -n 50

# 快速统计状态码分布
grep '"path":"/healthz"' probe.log | jq '.status' | sort | uniq -c
```

如果日志量大，建议按时间窗口切片，或使用 `awk` 聚合延迟：

```bash
grep '"path":"/healthz"' probe.log | jq -r '[.timestamp, .latency_ms, .status, .upstream] | @tsv' \
  | awk '{if($3>=500) print $0}'
```

### 3. 解读关键字段，找到隐藏的不健康模式

健康检查日志中值得关注的字段：

- `status`：200 之外的值（503、504、499 等）表示本次检查失败。
- `latency_ms`：突然增大的延迟可能暗示下游 Agent 或数据库变慢，即便最终返回 200，也应预警。
- `upstream`：显示真实检查的目标地址。如果 healthz 只检查 Gateway 自身进程，`upstream` 可能为 null；如果配置了“级联检查”，这里会显示后端 Agent 的地址和端口。**健康检查通过但 upstream 反复出错，是日志里最有价值的异常模式之一。**
- `error`：连接拒绝、超时、TLS 握手失败等具体原因。JSON 中的 `error` 字段不要忽略，很多 503 的真正根因就藏在里面。

一个典型异常行：

```json
{"timestamp":"2025-01-22T10:32:11Z","path":"/healthz","status":503,"latency_ms":5120,"upstream":"agent-2:8080","error":"dial tcp 10.0.1.15:8080: i/o timeout"}
```

说明健康检查依赖了 agent-2，而该 Agent 已经超时，导致整个节点被标记为不健康。根本问题是 Agent 故障，而非 Gateway 自身。

### 4. 提升日志可用性：诊断模式与采样

如果默认信息量不够，可以临时提升日志级别或开启诊断模式，重启服务后立即重现问题。在 OpenClaw Gateway 中，可以增加环境变量：

```bash
export OPENCLAW_LOG_LEVEL=debug
export OPENCLAW_PROBE_VERBOSE=true
```

这会输出健康检查的完整请求链、DNS 解析时长、连接建立耗时等细节。**注意：生产环境慎用，可能产生大量日志且轻微增加延迟。** 建议配合日志采样，只记录非 200 或延迟超过阈值的请求：

```yaml
probe:
  sampling:
    success_rate: 0.05      # 仅 5% 成功请求被记录
    error_rate: 1.0         # 所有失败都记录
    latency_threshold_ms: 500
```

## 踩坑点

- **把探活日志当无意义的噪声，直接关闭或丢弃**。在很多事故复盘里，发现根本原因就在被忽略的 503 日志中，但因为只收集了 error 级别，这些包含关键上下文的信息全丢了。
- **健康检查端点过重，自己把自己打挂**。如果 `/healthz` 里执行完整的 Agent 健康检查、甚至触发一次 MCP 工具调用，就可能在流量洪峰时让 Gateway 耗尽连接池，导致所有健康检查超时，进而被 K8s 重启，陷入死循环。**健康检查必须是 O(1) 的轻量操作**，最多只做一次上游 TCP 拨测。
- **probe 日志与业务 access log 混在一起，没有区分标识**。排查时难以快速分离。建议用 `log.probe.destination` 输出到独立文件或标准错误，或者在 JSON 里增加 `"type":"probe"` 字段。
- **多实例下日志分散**。如果使用 `kubectl logs` 只能看一个 Pod，需要聚合所有实例来判断是单个节点问题还是全局故障。快速方法：`kubectl logs -l app=openclaw-gateway --all-containers --tail=1000 | grep "/healthz" | jq...`

## 可复用建议

1. **健康检查端点设计**：`/healthz` 返回自身状态，`/ready` 返回是否就绪（依赖检查放在这里），`/live` 只做进程探活。禁止在 liveness probe 中检查外部依赖，防止级联重启。
2. **日志结构化 + 区分类型**：始终使用 JSON 输出，并在日志中显式标记 probe 类型，便于生产环境日志管道过滤与路由。
3. **告警规则**：基于健康检查日志设置延迟和失败率告警，例如 5 分钟内 `/healthz` 的 P99 延迟超过 2 秒或失败率 > 1% 触发警告。
4. **保留原始日志**：即使有集中式日志平台，也建议在本地保留最近 1 小时的 probe 日志文件，方便排障时快速 `grep`，不受平台查询延迟或索引延迟影响。
5. **定期演练**：在非生产环境故意停掉一个后端 Agent，观察健康检查日志的变化，验证团队是否能第一时间通过日志定位，而不是先去看仪表盘。

## 总结

OpenClaw Gateway 的健康检查日志不是“成功就可以忽略”的流水账，而是一个持续暴露系统依赖状态的窗口。把 `.status` 从 200 换成 503 的那一行，往往比任何监控曲线都更早地告诉你：某个 Agent 正在悄悄失效。学会看这行日志，就能在自动化体系尚未完全崩塌前，把问题拦在入口。

下次在午夜被报警叫醒时，不妨先 `grep "/healthz"`。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/fdc95c1df069dd9c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/cc42c12944d118cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/6ac7601a63fb20a6.png)

