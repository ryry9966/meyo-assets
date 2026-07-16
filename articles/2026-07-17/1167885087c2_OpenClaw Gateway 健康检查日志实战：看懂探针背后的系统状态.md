---
title: OpenClaw Gateway 健康检查日志实战：看懂探针背后的系统状态
feedId: 29338
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

在基于 OpenClaw Gateway 构建的 Agent 工作流中，健康检查（health check）是保证上游调度器、MCP 客户端或插件宿主正确分配流量的第一道防线。然而，很多团队只关注健康端点的 HTTP 状态码，一旦返回 200 就认为一切正常。当网关因依赖故障、配置漂移或资源争抢导致降级时，最先暴露问题的往往是那些被忽略的健康检查日志。

这篇文章整理我们在生产环境中阅读 OpenClaw Gateway 健康检查日志的实践方法，重点解决三个问题：日志里到底该看什么、如何从片段还原系统健康全貌、以及怎样避免常见的误判。

## 问题场景

OpenClaw Gateway 默认暴露两类探针：

- **Liveness probe**（存活探针）：`/healthz`，仅表示进程存活，不包含依赖状态。
- **Readiness probe**（就绪探针）：`/readyz`，包含关键依赖（如 MCP server、policy store、缓存后端）的连接状况。

典型错误如下：

1. 只看 `/healthz` 日志，但进程活着不代表能正常路由。
2. 日志级别设为 `info` 时，健康检查日志大量刷屏，淹没了真正的连接超时告警。
3. 出现间歇性的 `503` 后自动恢复，却没有关联到某一依赖的短暂不可用。
4. 将探针超时时间配置得过于宽松，导致调度器把流量发到已经僵死的实例上。

## 如何读懂一条健康检查日志

OpenClaw Gateway 的结构化日志中，健康检查组件标识为 `component=health`。关键字段如下（以 JSON 行输出为例）：

```json
{
  "ts": "2025-03-07T10:21:05.432Z",
  "level": "info",
  "msg": "health check completed",
  "component": "health",
  "probe": "readiness",
  "status": 503,
  "duration_ms": 3120,
  "dependencies": {
    "mcp-server-1": {"status": "timeout", "latency_ms": 3001},
    "policy-store": {"status": "ok", "latency_ms": 12},
    "cache": {"status": "ok", "latency_ms": 3}
  },
  "reason": "dependency_timeout"
}
```

**需要重点关注的点：**

- `probe`：区分是存活还是就绪。存活探针出现异常意味着进程内部死锁或资源耗尽，必须立即告警。
- `status`：就绪探针返回非 2xx 时，`dependencies` 字段会给出具体故障项。
- `duration_ms`：健康的就绪检查通常在 200 ms 以内。若接近或超过 `timeoutSeconds`，说明某个依赖正在临界状态。
- `dependencies` 中的细分状态：除了 `ok` 和 `timeout`，还可能出现 `connection_refused`、`tls_error`、`auth_failure`，各自对应不同方向的排障。

## 步骤化排查路径

1. **过滤核心日志**
   ```bash
   cat gateway.log | jq 'select(.component=="health" and .probe=="readiness" and .status!="200")'
   ```
   只拉取非健康的就绪检查，避免被成功日志淹没。

2. **分析故障时间线**
   将上述日志按 `ts` 排序，观察故障是否集中在某些时间段。结合网关所在 Pod 的资源监控（CPU throttling、内存 OOM 记录），判断是依赖端问题还是网关自身瓶颈。

3. **拆分依赖链路**
   若 `dependencies` 显示 `mcp-server-1` 持续 `timeout`，不要立刻重启网关。先单独从网关侧手动测试该 MCP server 的连通性：
   ```bash
   curl -v http://mcp-server-1:8080/health
   ```
   如果同样超时，问题在 server 侧；如果正常，则需要检查网关与该 server 间的网络策略或连接池配置。

4. **关注日志级别切换**
   在 `warn` 或 `error` 级别下，OpenClaw Gateway 会对连续多次失败的探针产出 `component=health`、`level=warn` 的日志，并附带回溯的失败计数。这比单独一条 `503` 更有价值，可以用它来触发告警聚合。

## 踩坑点

- **探针成功但依赖被降级**：某些依赖不可用时，网关可能退化到本地缓存或默认应答，就绪探针仍返回 `200`。记得让就绪探针明确枚举必需依赖，并在启动配置中设置 `health.check.strict-mode=true`，避免隐性降级。
- **健康检查日志的日志量陷阱**：生产环境中，存活和就绪探针通常每 10 秒执行一次，一个 3 副本的网关每分钟会产生约 18 条日志。若开启 `debug` 级别，会将每次依赖握手信息也输出，极易撑满日志存储。建议仅在排障时临时启用 `debug`，并通过采样工具只保留异常日志。
- **防火墙或 Service Mesh 侧车影响**：Sidecar 可能会拦截探针请求，导致日志中 `duration_ms` 虚高，但 dependencies 显示正常。需要区分是应用层耗时还是网络栈额外开销，可以通过网关内部的 `/debug/pprof` 抓取 goroutine 确认。
- **误用存活探针检测依赖**：绝对不要在 `/healthz` 里做依赖检查。这会让 kubelet 在依赖抖动时频繁重启网关进程，反而放大故障。

## 可复用建议

- **自定义健康检查配置**：在网关配置文件中显式指定每个依赖的检查端点、超时、重试次数，并暴露为独立的指标。例如：
  ```yaml
  health:
    readiness:
      dependencies:
        - name: mcp-server-1
          endpoint: http://mcp-server-1:8080/api/health
          timeout: 2s
          retries: 1
  ```
- **日志告警规则**：基于结构化日志字段设置告警，例如连续 3 次 `status >= 500` 且 `probe=readiness` 时触发；或某个依赖持续 `timeout` 超过 5 分钟时告警。避免只依赖指标而忽略日志中携带的故障原因。
- **构建整体看板**：将健康检查日志与网关的运行时指标（如 `upstream_rq_pending`、连接池饱和度）叠加，可以快速定位是“依赖全挂”还是“连接池耗尽”等不同根因。
- **日志保留策略**：按 `component=health` 单独输出到独立文件或索引，保留 7 天，方便回溯间歇性故障。

## 总结

OpenClaw Gateway 的健康检查日志远比一个 HTTP 状态码丰富。真正用好这些日志，需要养成三个习惯：区分存活和就绪的语义、读懂依赖字典里的异常状态、用过滤和聚合代替逐条扫描。当出现路由异常但就绪探针未触发时，往往是隐性降级或依赖检查覆盖不足，这时就要回头审计健康检查配置的严格性。

把健康检查日志当作系统的脉搏记录，而不是监控告警的冗余备份，能帮助团队在 Agent 链路出现性能退化之前就抓到早期信号。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-17/89939e5a9a790a4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-17/05af11d0d7792ff9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-17/fc1bc6e04b54caad.png)

