---
title: OpenClaw Gateway 健康检查日志：从被噪声淹没到一眼定界
feedId: 30009
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景：你为什么要盯着健康检查日志

OpenClaw Gateway 作为请求入口，负责将外部调用路由到后端的 Agent 运行时、MCP 工具节点以及自定义插件服务。为了保证路由表不会把流量导向已经挂掉的实例，Gateway 会以固定间隔对每一个上游端点发起健康检查（health check）。

在调试“为什么我的工具突然不可用”或“为什么路由总是打到旧实例”这类问题时，健康检查日志往往比业务日志更能直接给出定性结论。但默认配置下，这些日志量非常大，混在 debug 输出里极容易淹没真正异常的信号。如果你只是简单 `grep unhealthy`，很可能漏掉关键时间窗口内的软失败状态。

## 问题：一条日志里究竟哪些字段说了算

一条典型的 Gateway 健康检查日志看起来长这样（裁剪自实际 json 格式输出）：

```json
{
  "ts": "2025-03-29T14:32:11.234Z",
  "component": "health-checker",
  "endpoint": "mcp-tool/tavily-search:8001",
  "type": "http",
  "status": "unhealthy",
  "duration_ms": 5012,
  "consecutive_failures": 3,
  "error": "context deadline exceeded"
}
```

新手常犯的两个错误：  
1. 只看 `status` 字段，忽略 `consecutive_failures` 与 `duration_ms` 的组合。一次偶发超时可能只把 `consecutive_failures` 推到 1，还没达到摘除阈值，路由仍然有效，此时不应该盲目重启服务。  
2. 不区分健康检查类型。OpenClaw Gateway 支持 `http`、`grpc`、`exec`（插件自定义探活）三类，`error` 字段在不同类型下的含义完全不同。`grpc` 健康检查的超时错误往往表现为 `Unavailable` 或 `DeadlineExceeded`，而 `exec` 的失败只是脚本返回非 0，需要结合脚本输出进一步分析。

更隐蔽的问题是“假阴性”——服务明明存活，但网关持续判定为 unhealthy。这种情况多见于健康检查端点内部逻辑过重（例如联表查询数据库状态），导致延迟超过 `timeout` 设置，日志里的 `duration_ms` 会和 `timeout` 成对出现。

## 做法与步骤：从日志噪声中提取有效信号

### 第一步：确认你看到了什么级别

默认日志级别 `info` 只记录健康检查状态变更（healthy ↔ unhealthy）和首次失败；`debug` 级别则会输出每一次探活的结果。调试阶段建议临时打开 `debug` 几分钟，抓取完整序列，然后立刻调回 `info`，避免生产日志膨胀。

修改方式：Gateway 运行参数加 `--log-level debug` 或在配置文件 `logging.level.health-checker=debug`。

### 第二步：按时间线重建状态变更

用 `jq` 过滤出指定端点的状态变迁：

```bash
cat gateway.log | jq 'select(.component=="health-checker" and .endpoint=="<target>") | {ts, status, consecutive_failures, error}' 
```

关注点：  
- `consecutive_failures` 是否在 `failure-threshold`（默认 3）边缘反复横跳。这就是路由抖动的原因——实例时好时坏，但从未被真正摘除，导致部分请求失败。  
- `status` 的变化是否与 `duration_ms` 的峰值耦合。如果每当 `duration_ms > timeout` 时就出现 unhealthy，而服务端自身指标正常，说明是网络或网关到服务端的链路抖动。

### 第三步：对照健康检查参数判断根因

需要核对的配置项（通常在 gateway 配置文件的 `health_checks` 段）：

- `interval`: 探活间隔，默认 30s  
- `timeout`: 单次探活超时，默认 5s  
- `failure_threshold`: 连续失败多少次后将端点标记为 unhealthy，默认 3  
- `success_threshold`: 从 unhealthy 恢复到 healthy 所需的连续成功次数，默认 1  

常见误配：`timeout > interval`。这会导致健康检查 goroutine 堆积，表现为日志中同一个端点的探活时间戳重叠，`duration_ms` 异常增大。正确做法是保证 `timeout` 严格小于 `interval`，留出缓冲。

## 踩坑点：三个让你半夜怀疑配置的案例

1. **MCP 工具的 SSE 端点被当成 HTTP 探活**  
   如果健康检查对 SSE 流端点直接发 HTTP GET 且期待 200，某些实现会保持连接打开而不返回，直到 `timeout` 触发。错误日志里 `error: "context deadline exceeded"` 会持续出现，但工具本身功能正常。修复：为 SSE 端点配置 `type: grpc` 或自定义 `exec` 探活脚本，只检查进程存在即可。

2. **插件服务的 Exec 健康检查未设置超时包装**  
   `exec` 类型默认没有超时限制，如果自定义脚本因为依赖外部资源而 hang 住，健康检查 goroutine 会永远阻塞。日志上该端点的更新会“消失”，既没有 healthy 也没有 unhealthy 的新记录。需要给脚本内部加 `timeout` 命令或使用 `context` 控制，并在配置里指定 `timeout`。

3. **网关自身高负载导致健康检查超时假警报**  
   在高并发下，Gateway 的运行时可能延迟执行健康检查任务。日志里会看到多个端点同时出现 `duration_ms` 轻微超过 timeout，随后迅速恢复。此时应该查看 Gateway 自身 goroutine 数量和内存使用，而非盲目重启所有上游。

## 可复用建议

- **结构化日志永远别关**：坚持输出 JSON 格式，即使你在终端看也保留 `--log-format json`，方便以后用脚本批量分析。  
- **为健康检查单独设置 Prometheus 指标导出**：Gateway 内置了 `openclaw_gateway_health_check_status` 和 `openclaw_gateway_health_check_duration_seconds` 指标，用指标看板代替人肉盯日志。  
- **告警规则应基于变化率而非单一状态**：例如 `consecutive_failures` 增长速率 > 1/5min，比“存在 unhealthy 端点”更有效，避免夜间误报。  
- **对于关键链路，采用双重探活**：在 Gateway 层做基本存活性检查的同时，在 Agent 侧暴露内部健康状态接口，由 Gateway 通过 `exec` 或更细粒度的 HTTP 路径获取（如 `/health/deep`），分层排查。

## 总结

健康检查日志不是“看一眼 unhealthy 就拉群”的摆设，而是一份记录了每一次目标状态迁移的时序证据。读懂它需要把 `consecutive_failures`、`duration_ms`、配置阈值以及探活类型放在一起看。一旦你建立起“重建时间线→对照阈值→区分假阴性”这个排查习惯，大部分“莫名不可用”的反馈都能在网关日志里直接找到根因，不需要再翻遍上下游服务日志拼凑故事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/b832a4f4566cac4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/18be4eae10fb3b5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/2c907331072e275c.png)

