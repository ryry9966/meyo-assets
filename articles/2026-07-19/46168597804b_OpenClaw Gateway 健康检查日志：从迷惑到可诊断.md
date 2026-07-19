---
title: OpenClaw Gateway 健康检查日志：从迷惑到可诊断
feedId: 29666
source: 综合讨论
publishedAt: 2026-07-19
---

在生产环境的 OpenClaw Agent 编排里，Gateway 往往是最容易被忽视、却又最先暴露异常的一层。它承接了所有 MCP 工具调用、插件请求和 Agent 上下文路由，负载均衡器、Kubernetes 探针、旁路监控都在反复访问健康检查端点。但很多团队的实际情况是：大家只在 Pod 起不来时才看日志，一旦服务跑起来，就不再关心健康检查到底在报什么。这篇文章从几个典型场景出发，梳理 OpenClaw Gateway 健康检查日志的阅读路径，希望能帮你少踩一些“明明还活着却被杀了”的坑。

### 背景：健康检查不止“通”和“不通”
OpenClaw Gateway 暴露了三类常见的健康检查端点：`/healthz`（存活）、`/readyz`（就绪）和 `/livez`（部分部署中的别名）。存活检查失败意味着进程已死或严重阻塞，编排系统会直接重启；就绪检查失败则代表 Gateway 暂时不能处理请求，流量会被摘除。问题就在于，许多团队将所有检查都绑在同一个 `/healthz` 上，或者把依赖后端 MCP Server 的探测写进存活检查，导致一个缓慢的模型后端就把整个 Gateway 拖宕了。

### 常见的日志迷惑
打开 Gateway 的日志，可能会看到类似这样的记录：

```
GET /healthz 200 2ms
GET /readyz 503 5001ms
```

第一种是存活检查正常，但就绪检查返回 503 且耗时超过 5 秒。这说明 Gateway 进程存活，但内部某些组件未就绪——很可能是某个 MCP 连接还在建链中，或者 Redis/消息队列初始化超时。如果你把就绪探针的超时设成 1 秒，这类延迟就会导致 Pod 被不停地摘除又加入，流量反复抖动。

另一种更隐晦的日志是：

```
GET /healthz 200 0.8ms
GET /readyz 200 320ms
```

虽然都是 200，但就绪检查的响应时间在几分钟内从 2ms 逐渐爬升到 300ms。这往往是 Gateway 本地缓存或连接池积压的早期信号，继续恶化就可能转为 503。如果只盯着状态码，就会错过这个预警窗口。

### 从日志到排障的步骤
**1. 确认探针配置与端点对应关系**  
先检查部署清单中 livenessProbe 和 readinessProbe 分别指向哪个端点。建议遵循惯例：
- livenessProbe → `/healthz`，只检查进程内事件循环是否存活，不查任何外部依赖。
- readinessProbe → `/readyz`，检查 Gateway 是否完成初始化且可接受请求，但也要注意依赖项的超时策略。

如果你发现两者共用同一个路径，立刻分离它，否则后续所有排障都会建立在混乱的语义上。

**2. 用聚合视角看时序**  
单条日志看不出趋势。用 `grep` 或 logcli 过滤出 `/healthz`、`/readyz` 的请求，按分钟聚合状态码和延迟。比如使用 Loki 的 LogQL：

```
{app="openclaw-gateway"} |~ "/readyz" | logfmt | unwrap duration | quantile_over_time(0.95,1m)
```

当 P95 延迟超过 readiness 探针超时阈值的 70% 时，就要介入排查，而不是等到超时重启。

**3. 读懂 503 背后的错误细节**  
Gateway 健康检查返回 503 时，通常会在日志中附带内部原因，例如 “backend mcp-tool-not-ready” 或 “shutting_down”。如果你看到大量 “no healthy upstream”，说明后端某个 MCP Server 的健康检查也没通过，Gateway 的 `/readyz` 关联了聚合健康。这时要去查该后端服务的探针日志，而非在 Gateway 上反复重启。

另一种是 “shutting_down” 信号，表示 Gateway 收到了 SIGTERM，正在优雅退出。如果出现频率过高，检查是不是 Pod 终止周期太短，导致旧实例还没完全排干，新实例又因就绪失败被节流。

### 踩坑记录
- **过早的 initialDelaySeconds**：如果 initialDelaySeconds 设得太短，Gateway 还未完成 MCP 连接初始化就被就绪检查扫到，直接触发重启死循环。建议给 initialDelaySeconds 留足启动时间，并观察首次日志中 “gateway fully initialized” 的耗时，取 1.2 倍作为初始延迟。
- **后端慢查询拉高 Gateway 探针延迟**：某个 Agent 插件的健康检查实际调用了下游大模型进行一次简单推理，导致 readiness 变得极慢。解决方式是让 Gateway 的就绪检查对后端只做连接测试，而非语义测试。
- **忽略日志级别的差异**：OpenClaw Gateway 的健康检查失败可以配置为 WARN 级别，而非 ERROR。如果告警规则只抓 ERROR，这些 WARN 就会被淹没，直到服务真的不可用才发现。

### 可复用建议
1. **标准化探针定义**：在团队内部约定 Gateway 的存活/就绪端点行为，写进 Helm Chart 或 Kustomize 的基础模板，避免每次手动配置。
2. **建立健康检查 SLI**：将 `/readyz` 的成功率与延迟纳入 SLO，尤其关注 503 率的跳动。可以用 Prometheus 指标 `openclaw_gateway_healthcheck_status_total`（若已暴露）做告警，而非仅依赖容器重启次数。
3. **日志结构化**：确保健康检查日志包含 `path`、`status_code`、`latency_ms`、`reason` 字段，便于在 Grafana 面板中过滤。如果还在用纯文本日志，尽快切到 JSON 格式。
4. **定期复盘慢启动**：每次变更依赖后端后，观察连续 3 次部署的就绪时间分布，一旦中位数增幅超过 30%，就要优化连接预热逻辑，防止生产绿蓝部署时瞬间流量断路。

### 总结
OpenClaw Gateway 的健康检查日志是系统健康度的第一手线索，而不是编排工具的无聊心跳。通过区分存活/就绪、关注延迟趋势、解读 503 具体原因，你可以把“服务莫名重启”变成“提前 10 分钟发现的连接池饱和”。下一次看到那一连串 `/healthz 200`，不妨多看几眼 `/readyz` 的细节——那里藏着真正值得优化的杠杆点。

---

