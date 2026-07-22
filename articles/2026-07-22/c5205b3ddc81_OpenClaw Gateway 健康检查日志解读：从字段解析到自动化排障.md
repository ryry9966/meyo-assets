---
title: OpenClaw Gateway 健康检查日志解读：从字段解析到自动化排障
feedId: 30105
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景
在基于 OpenClaw 搭建的 Agent-MCP-插件 架构中，Gateway 是所有请求的统一入口，也是自动化运维探活的第一道关口。多数部署者习惯将 Kubernetes liveness/readiness probe 直接指向 `/healthz` 或 `/ready`，只要返回 200 就认为系统健康。但生产环境中经常出现一种矛盾：健康检查端点持续返回成功，Agent 却报告无法连接某个 MCP Server，或者插件调用超时。问题往往藏在健康检查的日志细节里——Gateway 返回 200 不代表所有子组件都“真健康”，只代表它自认为可以继续接受流量。

OpenClaw Gateway 的健康检查模块在设计上允许分层探测，并将每个依赖组件（上游 MCP 服务、插件注册中心、配置源、外部缓存等）的检查结果以结构化日志形式输出。看懂这些日志，才有可能在自动化链路出现故障前准确介入。

## 典型问题
假设你配置了一条规则：Agent 通过 Gateway 调用名为 `pdf-parser` 的 MCP 服务。tcp 探测该 MCP 服务端口是通的，Gateway 的 `/ready` 返回 200，但实际请求却报 `connection refused`。检查 Gateway 日志会发现类似输出：

```
{"level":"info","module":"healthcheck","component":"mcp-upstream","status":"failed","error":"dial tcp 10.0.1.5:9002: connect: connection refused","ts":"2025-03-11T02:12:04Z"}
```

同时 `/ready` 仍然返回 200，因为 OpenClaw Gateway 默认的 readiness 策略可能设置为“至少一个上游可用”或退化为“自身进程存活即可”。日志中已经明确记录了失败，但开发者常常忽略结构化日志流，只盯着 HTTP 状态码。这就是“假健康”的根源。

## 做法 / 步骤

### 1. 开启健康检查详细日志
OpenClaw Gateway 默认 health check 日志级别为 `info`，但对于超时、重试等细节需要调整为 `debug`。在配置文件中设置：
```yaml
logging:
  level: debug
  modules:
    healthcheck: debug
```
重启 Gateway 后，每次探测都会输出类似以下的完整记录：
```
{"level":"debug","module":"healthcheck","check":"mcp-upstream","target":"pdf-parser:9002","timeout":"2s","status":"fail","latency_ms":2001,"error":"context deadline exceeded","ts":"..."}
```
### 2. 理解关键字段
每一条 healthcheck 日志都是 JSON 行，常用字段包括：
- `check`：探测类型，如 `mcp-upstream`、`plugin-registry`、`db-ping`、`config-reload`。
- `status`：`pass`、`fail`、`warn`。
- `latency_ms`：探测耗时，逼近或超过 timeout 是危险信号。
- `error`：失败原因，如 DNS 解析失败、TLS 握手拒绝、超时。
- `consecutive_failures`（debug 模式可能包含）：连续失败次数，用于触发熔断。

### 3. 定位“假健康”窗口
OpenClaw Gateway 的探测间隔（默认 10s）与 readiness 状态变更之间存在窗口。日志中同一 `check` 可能出现间断性 `fail`，但 Gateway 可能因为上一次探测成功而暂时将自身标记为 ready。通过观察连续的 fail 日志并结合 `consecutive_failures` 达到阈值的时间点，才能准确推算出组件真正不可用的时刻。可以在日志聚合系统中按 `check=mcp-upstream AND status=fail` 过滤，并绘制时间线。

### 4. 将日志与探活端点对齐
Gateway 提供了独立的 `/healthz`（进程存活）和 `/ready`（流量就绪）。若 `/ready` 的实现逻辑未包含全部关键依赖，就会造成“日志失败、端点 200”的现象。这时需要修改 Gateway 的就绪判定配置，例如要求所有 `mcp-upstream` 检查全部通过：
```yaml
readiness:
  required_checks:
    - mcp-upstream
    - plugin-registry
```
修改后，当日志中出现某个上游 fail 时，`/ready` 会立即返回 503，并附带失败原因。

## 踩坑点
- **日志缓存/采样**：Gateway 在高并发健康探测时会缓存状态，debug 日志可能被压缩输出。若发现日志中断，检查是否有 `log.sampling_initial` 配置，确保关键失败不被采样丢弃。
- **滚动日志丢失上下文**：如果仅查看最近的日志文件，可能刚好错过一段连续的失败日志。建议将 Gateway 结构化日志接入集中式系统（如 Loki、Elasticsearch），并保留至少 1 小时的上下文。
- **自定义健康检查未写日志**：部分团队会自行编写 shell 脚本探测，却只在退出码中反映结果，不输出任何日志。导致 Gateway 只是代理了探测结果，内部却没有任何记录。应改用 Gateway 原生支持的 gRPC/HTTP 探活并确保输出到标准错误。
- **探测超时与请求超时混淆**：Gateway 探测 MCP 服务的超时设置（如 2s）可能小于 Agent 实际调用时的超时（5s）。健康检查日志超时不代表接口不可用，可能是性能退化。需要区分“慢”和“断”。

## 可复用建议
- 为 OpenClaw Gateway 创建专门的 Grafana dashboard，通过 Loki 查询 `module=healthcheck` 日志，统计各 check 的失败率和延迟 P95。
- 在 Prometheus Alertmanager 中基于日志指标（通过 mtail 或 Loki ruler）配置告警：连续 3 次 `mcp-upstream` fail 则触发通知，而不是等到 Agent 报错。
- 将 `/ready` 的探测行为与 Kubernetes probe 对齐，并确保 readiness 逻辑覆盖所有核心依赖，从代码层面杜绝“假健康”。
- 写自动化测试时，模拟上游不可用场景，检查 Gateway 日志中是否正确输出 `fail` 记录，并验证 readiness 端点返回 503。

## 总结
OpenClaw Gateway 的健康检查日志不是摆设，而是自动化体系里的“黑匣子”。它告诉你每一次探测的真实结果、延迟和失败原因。不要满足于 HTTP 200，沉到结构化日志行里，才能把“看起来没问题”变成“确实没问题”。当 Agent 大规模接入后，任何一次上游抖动都可能引发连锁故障，提前把健康检查日志读透，就是在为整个系统买保险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/fbe2def0d4a5582b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/f3bcc71cc71ae1c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/36ef8d082e2f2449.png)

