---
title: OpenClaw Gateway 健康检查日志：从乱码到可观测的排障路径
feedId: 30157
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在 OpenClaw 的 Agent 链路里，Gateway 是所有插件调用、MCP 工具请求、自动化任务触发的唯一入口。我们依赖它的 `/health` 端点做存活检测，却常常忽略背后那组健康检查日志的真实价值——当 Gateway 返回 503、上游超时或插件心跳丢失时，最先发出信号的其实是日志，不是告警。

一个典型场景：自动化流水线在凌晨突然大量失败，排查发现 Gateway 健康检查接口仍返回 200，但实际代理的某个 MCP Server 早已不响应。翻开日志，看到大量 `health check failed: context deadline exceeded`，但没有上下文告诉你到底是哪个 probe 失败、阈值是多少、是否触发了熔断。这就是为什么需要学会看 OpenClaw Gateway 的健康检查日志。

## 问题定位：日志在说什么

OpenClaw Gateway 的健康检查日志由内置的 `HealthProbe` 模块产生，默认输出到 stdout，随主进程日志一起被收集。常见日志片段：

```
{"level":"warn","ts":"2025-03-18T09:12:33.211Z","msg":"health check slow","component":"plugin:webhook-relay","duration_ms":3120,"threshold_ms":3000}
{"level":"error","ts":"...","msg":"health check failed","component":"mcp:data-fetcher","error":"dial tcp 10.2.1.5:9090: i/o timeout","consecutive_failures":3}
```

这两行的区别很大：`slow` 表示单次探测耗时超过阈值但未失败，用于提前发现性能退化；`failed` 则代表探测已达失败判定标准，会触发对应组件的降级或重启策略。日志中的 `component` 字段是排障的核心——它能定位是内置插件、外部 MCP Server，还是 Gateway 自身依赖（如 Redis 连接池）。

如果没有健康检查日志，你只能从下游服务的超时错误反推，但那时可能已经错过最佳干预窗口。

## 做法：三步读懂并利用日志

### 1. 让日志可检索：结构化字段标准化

OpenClaw Gateway 默认使用 zap 输出 JSON 日志，但仍依赖配置文件开启健康检查详细程度。确认 `config.yaml` 中：

```yaml
health:
  log_level: debug          # 默认 info 不会输出单次探测成功日志
  probes:
    - type: http
      endpoint: /internal/health
      interval: 5s
      timeout: 3s
      failure_threshold: 3
      success_threshold: 1
```

设置 `log_level: debug` 后，每次探测的结果都会输出，包括成功时延、状态码、返回体长度。这对建立基线很有帮助。

对于生产环境，建议使用 `log_level: info`，但务必打开 `slow_threshold_ms` 和 `error` 级别的日志采集。同时，所有日志统一带上 `component`、`probe_type` 字段，便于日志平台过滤。

### 2. 建立观察基线：从成功日志开始

不要等故障了才看日志。先拉取一段正常期的健康检查日志，统计每个 `component` 的 `duration_ms` 分布：

```bash
cat gateway.log | grep "health check completed" | jq '{component, duration_ms}' | ...
```

你会发现某些插件天然就慢（比如需要查外部 API 的 Agent 插件），500 ms 是常态；而 MCP 本地进程往往在 10 ms 以内。这些基线值能帮你设定合理的 `slow_threshold_ms`，而不是一刀切的 1000 ms。

### 3. 构建排查链路：从日志到根因

当收到 `health check failed` 时，不要只看这一行。立刻向前翻同一 `component` 的日志，寻找“斜率”变化：是否之前出现过 `slow` 预警？`consecutive_failures` 是如何递增的？如果连续失败计数是 1、2、3 然后触发恢复，这可能只是网络抖动；如果是 3 次突然同时出现（同一秒），则更像节点宕机或端口被防火墙阻断。

另一个关键点是 `error` 字段的内容：
- `dial tcp ... i/o timeout` → 网络不可达或服务未监听
- `connection refused` → 端口未开启或服务进程崩溃
- `context deadline exceeded` → 探测超时，通常意味着上游服务负载过高
- `status 503` → 服务自身健康检查失败，需登入该组件查看

如果能将健康检查日志与 Gateway 自身的 metrics（如请求队列长度、goroutine 数量）交叉印证，定位速度会更快。这在 OpenClaw 中可以借助 Prometheus 指标实现，但日志永远是第一现场。

## 踩坑点

1. **以为 200 就是健康**：Gateway 健康端点仅代表自身进程存活，不代表所有 probe 通过。必须区分“网关存活”和“组件就绪”，否则会出现“监控全绿、业务全跪”的情况。
2. **日志量爆炸**：开启 debug 后，高频探测（如 1s 间隔）的日志可能淹没磁盘。解决方法是仅对生产环境开启 error 以上，或通过日志采样器限制每秒写入条数。我们实践使用 `log_level: info` + 单独输出 slow/error 日志的方式。
3. **忽略 cold start**：Gateway 刚启动时，所有插件并行初始化，健康检查可能集中失败，这容易掩盖真实故障。建议在告警规则中加入“启动窗口”排除策略，或在日志分析时过滤启动后前 30 秒的记录。
4. **组件标签缺失**：早期版本中，部分自定义插件的健康检查日志未自动注入 `component` 字段，导致无法区分。可以通过插件 SDK 提供的 `WithProbeName` 显式设置，确保所有路径都有唯一标识。

## 可复用建议

- **标准化日志格式**：所有团队自研的 Agent 插件都应当实现 `HealthChecker` 接口，并输出与 Gateway 一致的字段（`component`、`duration_ms`、`error`）。这能让运维同一条流水线处理所有组件的健康状态。
- **建立慢探测基线告警**：不要只对失败报警。设置一个比 P95 略高的 `slow_threshold_ms`，提前捕获数据库连接池耗尽、DNS 解析变慢等问题。
- **将健康检查日志接入事件时间线**：在故障复盘时，将 Gateway 日志与上游发布记录、配置变更对比，往往能发现“刚发布的新 MCP Server 未注册健康检查端点”这类低级错误。
- **编写日志解析脚本**：一个简单的 jq 脚本，可视化为文本表格：

```bash
cat health.log | jq -r 'select(.msg=="health check failed") | [.ts, .component, .error] | @tsv'
```

这比在 ELK 里翻页快得多。

## 总结

OpenClaw Gateway 的健康检查日志不是运维的“背景噪音”，而是分布式代理系统最直接的脉搏。理解它的结构、建立基线、构建从 slow 到 failed 的排查路径，会让你在半夜被电话叫醒时，少看 10 分钟屏幕就能点出故障插件。不要等到 503 才想起它。

---

