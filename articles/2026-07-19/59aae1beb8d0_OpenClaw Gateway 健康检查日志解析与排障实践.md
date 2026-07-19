---
title: OpenClaw Gateway 健康检查日志解析与排障实践
feedId: 29652
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

在 OpenClaw 的网关部署中，健康检查（health check）是保障 Agent、MCP 工具后端和插件的可用性的第一道防线。无论是反向代理到本地的 Python MCP Server，还是转发到远端的 Agent Runtime，配置 `/health`、`/ready` 端点并通过网关主动探测已是标配。问题往往不在配置本身，而在于：当健康检查状态异常时，从日志里快速定位根因比想象中更脏更累。很多同学的反馈都是：“日志里一堆 `health check failed`，但上下游明明都活着。”

本文围绕 OpenClaw Gateway（基于插件扩展的网关组件）的健康检查日志展开，梳理日志结构、常见故障模式、日志分析思路，以及可落地的告警和排障策略。内容适用于使用 OpenClaw 做 Agent 编排、插件网关、MCP 中继的工程化实践。

## 日志长什么样

OpenClaw Gateway 的健康检查日志默认写入 `stdout`，并携带 `component=healthcheck` 标签。一条典型记录长这样：

```json
{
  "time": "2025-03-12T09:23:41.221Z",
  "level": "warn",
  "component": "healthcheck",
  "target": "mcp-server:weather",
  "type": "passive",
  "result": "unhealthy",
  "status_code": 503,
  "latency_ms": 2310,
  "consecutive_failures": 3,
  "message": "health check failed: context deadline exceeded"
}
```

关键字段：

- `type`：`active`（网关主动探测）或 `passive`（根据实际请求流量判定，下文会展开）。
- `target`：被检查的上游标识，通常是服务名或插件名。
- `result`：`healthy` / `unhealthy` / `degraded`。
- `status_code`：探测返回的 HTTP 状态码，注意 2xx/3xx 一般计为健康，其余由策略定义。
- `consecutive_failures`：连续失败次数，达到阈值会触发摘除或降权。
- `message`：具体错误信息，如超时、连接拒绝、tls 错误等。

缺省日志级别是 `warn`，仅当状态变化或连续失败时才输出。这意味着**没有日志的时间段本身就是一种信号**：目标在此期间持续健康。

## 常见问题与排查步骤

### 1. 被动健康检查的“伪装”失败

OpenClaw Gateway 支持被动健康检查（passive health check），即利用真实业务请求的响应码和延迟来更新后端健康状态。这会带来一个反直觉的现象：**健康检查日志报 `unhealthy`，但主动探测（`/health`）始终返回 200**。原因往往是某个业务接口偶发 5xx 并触发了被动规则，而此时主动探测依然正常。

**排查做法**：

- 过滤 `component=healthcheck` 且 `type=passive` 的日志，观察 `status_code` 和 `consecutive_failures`。
- 如果被动失败伴随上游高延迟（`latency_ms` > 阈值），很可能是业务侧慢查询耗尽连接池，健康检查共享了同一连接队列导致超时。
- 临时修复：调大被动健康检查的 `failure_threshold` 或缩短统计窗口。
- 根治：拆分业务流量与健康探测的连接池，并对上游补限流和降级开关。

### 2. 启动窗口的“抖动误报”

插件或 MCP 服务冷启动时，端口监听早于依赖就绪（如模型下载、数据库连接），导致最初三次健康探测全部失败。日志中会出现连续的 `unhealthy` 和 `consecutive_failures` 迅速攀升至阈值，网关直接摘除节点，即便服务后续已就绪。

**排查做法**：

- 观察 `time` 字段与部署时间戳的间隔，若失败集中在启动后 5~30 秒内，大概率是预热问题。
- 解决方案：给上游服务增加 `startupProbe` 思路的 readiness 延迟，或在网关侧将 `initial_delay` 设置为略大于历史启动耗时。
- 另外，确保日志中 `message` 出现 `connection refused` 时，直接去 k8s Pod event 或容器日志里确认端口监听时间点。

### 3. 状态码“正常”但不健康

很多人只把 2xx 视为健康，实际情况更复杂。比如某些 Agent 端点返回 302 重定向到登录页，虽然状态码为 3xx，但服务实质不可用。OpenClaw Gateway 默认将 2xx 和 3xx 标记为健康。如果期望更严格，需要自定义健康判定规则。

**排查做法**：

- 从日志中抽取 `status_code` 分布：`jq '.status_code'` 或 `grep` 聚合。
- 若频繁出现 301/302/307，需检查健康检查端点是否被中间件重定向，应直接指向 `/health` 或 `/ready` 且禁掉外部认证。
- 配置策略：将 `health_statuses` 明确设置为 `[200, 204]`，避免无意义的 3xx 掩盖故障。

### 4. 超时日志中的“隐形瓶颈”

日志里的 `context deadline exceeded` 或 `i/o timeout` 往往只说明超时，没有指出是网络、序列化还是上游逻辑慢。此时需要结合 `latency_ms` 和上游自身日志联合定位。

**做法**：

- 在健康检查日志里提取 `target` 和 `latency_ms`，倒入时序库（如 VictoriaMetrics）绘制 p95/p99 延迟曲线。
- 当延迟升高但上游 CPU 不高时，大概率是网关到上游的网络出现了重传或丢包。可以启用网关的 tcp keepalive 并缩短超时。
- 若上游为 Python MCP Server，注意其健康检查接口内是否有同步阻塞调用（如不必要的大模型 ping），把简单探活扩展成重操作是常见的坑。

## 可复用建议

1. **结构化日志上报警**  
   将 `component=healthcheck` 且 `result=unhealthy` 的日志接入 Alertmanager。告警规则不应仅凭一次失败触发，建议使用 `consecutive_failures > 2` 且持续时间超过 60 秒。

2. **主动+被动双视角看板**  
   在 Grafana 面板中同时展示主动探测结果和被动健康检查失败率，避免因被动失败而误判全量故障。

3. **日志保留与回放**  
   健康检查日志体积通常很小，保留 30 天可助力事后归因。利用 `jq` 或 `lnav` 对 JSON 日志做时间窗口统计，能更快发现周期性故障（如定时任务导致的雪崩）。

4. **健康检查端点约定**  
   对所有 Agent、MCP 和插件强制约定 `/health` 必须是轻量级、无副作用、无认证的，仅返回 `{"status":"ok"}` 和 200。Readiness 单独用 `/ready`，避免混淆。

## 总结

健康检查日志远不止“失败—告警—修复”这么直白。从被动检查的侧信道故障，到启动抖动、状态码误判、超时迷雾，每一行 `unhealthy` 背后都可能藏着不同的根因。把日志当作可观测性的一等公民，结构化存储、多维聚合、对齐上下游时间线，才能在 OpenClaw Gateway 的复杂调用拓扑中，把健康检查从一种心跳升级为真正的排障罗盘。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/797232f98645b73f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/2f10f41cbcfddfc8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/891c456784f94bd5.png)

