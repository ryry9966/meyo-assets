---
title: OpenClaw Gateway 健康检查日志：从“看不懂”到高效排障
feedId: 29344
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

在 OpenClaw 体系的日常运维中，Gateway 是所有 Agent、MCP 服务、插件和自动化流量的唯一入口。健康检查（health check）是 Gateway 判断上游服务是否就绪的核心机制——它周期性探测配置的 endpoint，决定是否将请求路由到对应后端。

一旦某个服务被标记为 unhealthy，用户侧就会出现 502 或超时，而排查的第一站几乎永远是 Gateway 的日志。但很多实践者面对大量 `health check` 相关行时，往往抓不住关键信息，排查方向跑偏。本文还原一个真实工程场景：如何读懂 OpenClaw Gateway 的健康检查日志，并快速定位根因。

## 问题

最常见的误区是：**“日志说 unhealthy，就是服务挂了”**。实际上，Gateway 日志中健康检查的输出包含多种中间状态：连接超时、读超时、TLS 握手失败、HTTP 状态码不匹配、连续失败计数器累积等。日志量一大，容易把临时的网络抖动误解为服务宕机，或者忽略了配置不合理导致的误判。

另一个问题是，部分人习惯直接跳到 error 日志查看，但健康检查的成败信息往往出现在 info 或 debug 级别，若未启用详细日志，甚至连失败原因都看不到。这让排障变得像猜谜。

## 关键日志字段与解读步骤

### 1. 确保日志级别生效
在 OpenClaw Gateway 配置（如 `gateway.yaml`）中，需要将健康检查相关组件的日志级别调至 `info` 或更细粒度：
```yaml
logging:
  level: info
  component_levels:
    health_checker: debug
```
开启后，每次探测都会输出一条结构化日志。

### 2. 典型日志行示例
```json
{
  "time": "2025-03-07T14:23:01.234Z",
  "level": "debug",
  "component": "health_checker",
  "upstream": "mcp-server-1",
  "check_type": "http",
  "uri": "http://10.2.1.5:9000/health",
  "status": "unhealthy",
  "status_code": 0,
  "error": "dial tcp 10.2.1.5:9000: connect: connection refused",
  "consecutive_failures": 3,
  "healthy_threshold": 1,
  "unhealthy_threshold": 3,
  "message": "health check failed"
}
```
这条日志已经包含了排障所需的绝大部分信息。我们拆解：

- **`status`**：`unhealthy` 表示本次探测失败。但要看 `consecutive_failures` 是否已经达到 `unhealthy_threshold`（此处为 3），若未达到，Gateway 仍会把服务视为 healthy（即“不健康”状态尚未生效）。
- **`status_code`**：0 表示根本没有收到 HTTP 响应，说明连接层面出了问题，而 `error` 字段直接指出 `connection refused`，意味着目标端口没有进程监听。
- **`error`**：切勿忽略。这是具体原因，例如 `context deadline exceeded`（超时）、`x509: certificate signed by unknown authority`（证书问题）等。

### 3. 区分瞬断与真宕机
如果日志中连续失败刚好达到阈值，随后马上又恢复为 `healthy`，且 `error` 为 `i/o timeout`，大概率是服务偶尔处理请求慢导致探测超时，而非真正崩溃。此时应检查超时配置与上游负载，而不是直接重启服务。

若 `consecutive_failures` 持续增长并远超阈值，同时 `error` 固定为 `connection refused`，则基本可判定服务已停止或端口未绑定。

### 4. 状态转换日志
在达到 unhealthy 阈值时，Gateway 会输出关键事件：
```json
{
  "level": "warn",
  "component": "health_checker",
  "upstream": "mcp-server-1",
  "message": "upstream marked as unhealthy after 3 consecutive failures",
  "new_status": "unhealthy"
}
```
同样，恢复时会输出 `marked as healthy`。这些是告警和自动化策略的触发器，也是日志回溯时的锚点。

## 踩坑实录

**坑1：threshold 设置过小，日志里“狼来了”**  
有人为了灵敏，把 `unhealthy_threshold` 设为 1，任何一次探测超时就立刻摘除节点。结果网络微小波动导致频繁切换，日志刷屏。建议根据服务响应时间 P99 值设定，通常 3~5 次是一个经验值。同时，`interval` 和 `timeout` 要配合：timeout 不应大于 interval，否则探测会排队堆积。

**坑2：只看 unhealthy，不看 passive health check**  
部分配置允许 passive health check（基于真实请求的失败计数），其日志与主动探测混在一起，需要根据 `check_type: "passive"` 区分。排查时若只关注主动探测，会漏掉由 5xx 错误触发的摘除。

**坑3：日志结构化但未采集，只能用 grep**  
单机排查尚可，一旦多实例，纯 grep 效率极低。如果还没有集中日志平台，至少将日志输出为 JSON，通过 `jq` 快速过滤：  
```bash
cat gateway.log | jq 'select(.component=="health_checker" and .upstream=="mcp-server-1" and .status=="unhealthy")'
```

**坑4：证书过期被误判为服务宕机**  
当错误是 `x509: certificate expired` 时，服务本身可能正常，但 TLS 握手失败。此时日志中的 `status_code` 也是 0，容易被归为连接失败。务必检查 error 字符串。

## 可复用建议

1. **规范日志字段**：确保所有后端健康检查使用统一的 `component` 值（如 `health_checker`），便于筛选。
2. **设定合理的探测参数**：`timeout` 基于 P99 延迟 × 1.5，`interval` 大于 timeout，`unhealthy_threshold` ≥ 3。
3. **与 metrics 联动**：导出的 `unhealthy` 状态变化作为 Prometheus 指标，可以在不翻日志的情况下快速发现异常。
4. **构建诊断查询模板**：保存几个常用查询：
   - 查看某个 upstream 最近 30 次健康检查状态：`jq 'select(.upstream=="xxx") | {time,status,error}'`
   - 筛选所有 unhealthy 事件的 transition 日志：`jq 'select(.component=="health_checker" and .new_status=="unhealthy")'`
5. **保留错误详情**：生产环境下，不要将 error 字段缩写，它是第一手证据。

## 总结

OpenClaw Gateway 的健康检查日志并不复杂，但需要掌握解读框架：先看 `status` 和 `error` 判断是连接层还是应用层问题，再通过 `consecutive_failures` 与阈值对比确认是否真正触发 unhealthy，最后结合 transition 日志建立时间线。避开阈值过小、忽略错误字符串等常见坑，配合结构化查询，基本可以做到秒级定位。下次再遇到“明明服务没挂，Gateway 却报 unhealthy”时，不妨直接打开 debug 日志，真相往往就藏在一行行 `health checker` 输出里。

---

