---
title: OpenClaw Gateway 健康检查日志怎么读：从“绿了就行”到真正可观测
feedId: 30179
source: 综合讨论
publishedAt: 2026-07-23
---

在 OpenClaw 的日常运维里，健康检查日志大概是最容易被“扫一眼就过”的部分。`/health` 或 `/ready` 返回 200，Dash 面板亮绿灯，很多人就觉得万事大吉。直到有一次上游模型服务已经半死不活，网关还在把流量往里打，才发现健康检查日志远不止“成不成功”这一个维度。

这篇帖子的背景是：我们在内部用 OpenClaw Gateway 做多模型路由和 MCP 插件入口，健康检查端点同时绑定了 PostgreSQL、Redis、部分 Agent 运行时和对外依赖的第三方 API。问题出在一次毫秒级抖动——所有端点都返回了 200，但实际已有两个上游进入“半开放”状态，因为健康检查的超时设置和重试策略掩盖了真实延迟上升。

下面直接拆开看，怎么把 OpenClaw Gateway 的健康检查日志读透。

---

### 1. 先确认日志打在哪里

OpenClaw Gateway 默认将健康检查结果以结构化 JSON 写入 stdout，通常由 filebeat 或 vector 采集。如果没有做额外配置，日志级别需要至少为 INFO 才能看到所有探活记录。很多人会把健康检查的日志级别单独提升到 DEBUG，这在排查问题时非常有用，但平时会产生大量噪音。我的建议是：**保持 INFO 级别记录整体结果，对失败或延迟异常的单次检查单独输出 WARN**。配置如下：

```yaml
logging:
  level:
    root: INFO
    com.openclaw.gateway.health: DEBUG  # 排查时开启
```

健康检查日志的关键字段：

- `check_name` — 检查项名称（如 `db`, `redis`, `agent_runtime`）
- `status` — `UP` / `DOWN` / `DEGRADED`
- `duration_ms` — 本次检查耗时
- `error` — 失败时的异常信息
- `details` — 各子检查明细

如果日志里只有 `status: UP`，没有 `duration_ms` 和 `details`，那说明你只看到了“汇总结果”，而这正是踩坑的起点。

---

### 2. 区分瞬时失败、持续降级与“假健康”

典型的一次故障排查流程：

当日志频繁出现：

```json
{"check_name":"redis","status":"DOWN","error":"connection timeout","duration_ms":3002}
```

你很容易立刻去查 Redis。但等一下—— `duration_ms` 恰好等于你的超时阈值（假设 3000ms），并且同一时刻 `db` 检查也出现 1200ms 延迟。这时候需要把几类日志放到时间轴上看：

- **瞬时网络抖动**：单个检查 DOWN，下一次立刻恢复，error 为 `connection reset` 或 `i/o timeout`，且其他检查无延迟波动。这种一般不用告警。
- **依赖资源饱和**：Redis 检查延迟从 2ms 升到 800ms，但还没有超时。状态仍然是 UP，可实际上 Redis 的慢查询已经在堆积。只看状态就会漏掉。
- **假健康**：健康检查只做了 `PING`，或者只检查连接池能否借出连接，却没有真正执行一次读写。这在第三方 API 的检查里最常见——对方可能直接缓存了健康检查路由，永远返回 200。

对抗假健康的方法：健康检查不要走特权路由。OpenClaw Gateway 支持自定义检查逻辑，比如对 PostgreSQL 执行 `SELECT 1` 并断言返回，而不是仅仅看连接池状态。

---

### 3. 三个必看指标

除了读日志文本，我会在 Grafana 上对健康检查日志聚合三个指标，一旦异常立刻回过头翻原始日志：

- **健康检查成功率（按检查项）**：低于 99.9% 就要开始看 error 分布。
- **P95/P99 延迟（duration_ms）**：比成功率更敏感。出现翻倍但成功率不变，往往是性能瓶颈的前兆。
- **降级状态（DEGRADED）出现次数**：OpenClaw Gateway 支持部分检查失败时标记为 DEGRADED 而非 DOWN。这个状态如果长时间存在，说明你允许的半开运行实际上已经在拖着整体吞吐量下降。

---

### 4. 踩坑：熔断器与健康检查的互相干扰

一个非常隐晦的问题：我们曾经在网关层对上游 API 配置了熔断器，熔断触发后，所有调用快速失败。但健康检查端点因为使用了同一个连接池/客户端，也被熔断器拦截，直接返回 `DOWN`。于是健康检查日志告诉我们“上游挂了”，实际上只是网关自己的熔断器打开了。错误信息里只有 `CircuitBreakerOpenException`，如果不熟悉调用链路，会误以为对方真的宕机了。

解决方法：**为健康检查单独配置一个不受熔断器影响的轻量客户端**，或者健康检查走独立的探测端点。此外，在日志里区分“熔断导致的失败”和“真正的连接失败”，这一点可以通过在 `error` 字段里加入错误码或异常类型实现。

---

### 5. 可复用的建议

- **结构化日志 + jq 是排障利器**：如果日志量大，用 `jq 'select(.check_name=="agent_runtime" and .duration_ms>500)'` 快速过滤出慢检查。
- **自定义健康检查不要只做连通性**：至少包含一次真实的读操作，最好还有写操作的影子表（并回滚）。
- **告警规则不要只盯 DOWN 状态**：DEGRADED 或连续 3 次 duration_ms 超过基线的 3 倍，应该触发低级告警。
- **日志里带上 trace_id**：如果健康检查触发深度依赖（比如通过 Agent 调了一次 MCP server），把 trace_id 打进去，后续可以关联全链路追踪。

---

### 总结

OpenClaw Gateway 的健康检查日志像个沉默的诚实人——它告诉你够多，但前提是你得问对问题。从“有日志就行”进化到“用日志做趋势判断和依赖可观测”，中间的区别往往是避免一次凌晨三点故障的关键。

不要只盯着绿色小勾。Duration、error 类型、降级信号，这三样东西才是健康检查日志真正的“体检报告”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/4d5841e3deca733d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/207828e8288c2487.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/6f407385e59d8d4b.png)

