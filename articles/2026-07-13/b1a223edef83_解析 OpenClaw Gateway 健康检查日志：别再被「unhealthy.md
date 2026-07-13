---
title: 解析 OpenClaw Gateway 健康检查日志：别再被「unhealthy」吓到
feedId: 28894
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景：为什么 Gateway 健康检查日志总是被误读

在 OpenClaw 的部署拓扑里，Gateway 是所有外部请求进入 Agent 集群的入口。它承担着流量路由、协议转换、安全校验和健康探测的职责，尤其在与 MCP 插件及各种 Agent 后端对接时，健康检查几乎是一个“无感但致命”的组件。

开发者最常见的困惑是：明明服务一切正常，Gateway 日志里却频繁出现 `health check failed`；或者监控面板上短暂红闪，点进日志只能看到一句 `liveness probe: connection refused`，难以还原现场。更麻烦的是，当插件端只暴露了一个非标准的健康端点，Gateway 的探测日志就成了唯一的排障线索，但多数人仅会简单搜索 “unhealthy”，错过了关键上下文。

这篇文章不会复述 Kubernetes 或负载均衡器的通用健康检查概念，而是聚焦在 **OpenClaw Gateway 自身的健康检查日志字段、误报根因和一套可复用的排查步骤**上。

## 问题：读不懂日志，排查只能靠重启

一个典型的 OpenClaw Gateway 部署会同时承载两类健康检查：

- **Gateway 自身上报的 liveness/readiness**（通常暴露在 `:8080/healthz` 和 `:8080/readyz`）
- **代理后端（Agent 进程、MCP 插件服务）的 health check**，由 Gateway 根据配置主动探测

而日志中引发困惑的，绝大多数来自后一种——Gateway 作为探测器，尝试访问后端服务的健康端点，并将结果以结构化 JSON 写入 stdout。初看这段日志：

```json
{"ts":"2025-03-13T10:11:02Z","level":"info","component":"health-checker","target":"agent-vision:6060","checker":"liveness","result":"unhealthy","reason":"dial tcp 10.2.1.5:6060: connect: connection refused","duration_ms":3}
```

很多人第一反应是“后端挂了”，于是重启 Agent，却发现重启前后服务其实一直在正常处理业务请求。这种假阳性往往来自探测路径、端口映射或网络策略配置失配，和进程死活无关。

## 做法：从日志字段到根因的三步走

### 1. 搞清日志中的关键字段

OpenClaw Gateway 的健康检查日志采用统一的 `component=health-checker` 标记，建议直接在日志聚合平台或 `jq` 里过滤：

```bash
cat gateway.log | jq 'select(.component=="health-checker")'
```

每个条目固定携带以下字段：

- `checker` ：`liveness` 或 `readiness`。前者判断进程是否挂掉，后者判断是否能够接收请求。
- `target` ：后端服务标识，通常是 `plugin-name:port` 格式，可以直接对应到 Gateway 配置中的 `backends[].name`。
- `result` ： `healthy` 或 `unhealthy`。
- `reason` ：失败时的具体错误信息，如超时、连接拒绝、状态码非 200、TLS 握手失败等。
- `duration_ms` ：探测耗时，用于发现后端的性能退化。

**重要区别**：如果是 `liveness` 持续 `unhealthy`，Gateway 会触发该后端的重启策略（若配置了自动恢复）；如果是 `readiness` 短暂 `unhealthy`（例如在部署滚动更新期间），则只是临时把流量摘除，不应过度反应。

### 2. 开启 debug 模式锁定探测端点

默认的 info 级别可能不显示实际请求的 URL 路径和返回的状态码。遇到无法解释的失败时，临时将 Gateway 的日志级别调整为 debug：

```bash
oc-gateway config set --component health-checker --log-level debug
```

再观察日志，会多出类似以下的条目：

```json
{"level":"debug","msg":"performing health check","url":"http://agent-vision:6060/healthz","timeout":"2s"}
{"level":"debug","msg":"health check response","status_code":404,"body":"404 page not found\n"}
```

这时候问题就清晰了：插件暴露的端点是 `/health`，而 Gateway 默认探测 `/healthz`，导致永久 404，却被抽象成了 `unhealthy`。这类路径不匹配是最常见的“乌龙”。

### 3. 验证插件侧的真实健康端点

对于 MCP 插件，没有统一的健康检查规范。需要直接查看插件文档或运行进程的命令行参数，确认是否包含 `--health-endpoint` 之类的设置。也可以在 Gateway 所在的网络空间直接 curl 验证：

```bash
# 从 Gateway Pod 内部发起（示例）
curl -v http://agent-vision:6060/health
```

如果得到 `200` 且返回 `{"status":"ok"}`，就可以确定是 Gateway 配置路径有误。

修正办法是在 Gateway 的 backend 定义中显式指定 `health_check_path`：

```yaml
backends:
  - name: agent-vision
    address: agent-vision:6060
    health_check:
      path: /health
      interval: 10s
      timeout: 3s
      unhealthy_threshold: 2
```

重启 Gateway 后，日志里 `result=healthy` 会立即出现。

## 踩坑点

1. **健康检查端口误差**  
   许多 MCP 插件监听的管理端口（如 `9090`）与业务端口（如 `6060`）不同。Gateway 配置中 `address` 可能带上了错误的端口，导致探测永远失败，但业务流量却通过另外的路由规则正确转发。排查时记得核对 `target` 字段中的端口。

2. **网络策略阻断 ICMP/Dial**  
   在 Kubernetes 环境下，如果 NetworkPolicy 只放行了业务端口，而健康检查使用了 TCP 建连（Dial），那即使 curl 能通，Gateway 的 go-health 库却可能因连接被直接拒绝而快速失败。日志中的 `reason` 会明确显示 `connection refused`。修复方法是确保网络策略允许 Gateway 的 Pod 到后端健康检查端口的 TCP 流量。

3. **超时设置不合理**  
   有些 AI Agent 的 readiness 需要加载模型，启动时间长达 30 秒。如果 Gateway 的 `timeout` 保持默认 2 秒，那么启动期间日志会一直刷 `unhealthy: context deadline exceeded`。这类情况需要区分启动期和运行期，为 `initial_delay_seconds` 留足余地。

4. **将健康检查日志直接用于告警**  
   频繁的 `unhealthy` 并不一定表示故障。例如滚动更新时，旧 Pod 被终止，Gateway 会在 `unhealthy_threshold` 次失败后摘除流量，这期间日志必然出现 `unhealthy`。正确的做法是基于持续 unhealthy 的窗口（如连续 3 次失败且 duration_ms 上升）或结合业务监控指标来触发告警，而不是简单地用关键字匹配。

## 可复用建议

- **统一健康检查规范**：在团队内部要求每个 MCP 插件必须暴露一个标准化路径（如 `/health`）并返回 `200` 和 `{"status":"ok"}` 的 JSON，同时在文档中注明。
- **结构化日志永远比 grep 可靠**：在 Grafana Loki 或 ELK 中基于 `component=health-checker` 和 `result=unhealthy` 创建索引，用 `target` 字段分组展示失败趋势，可以快速定位哪类后端不稳定。
- **建立健康检查自查清单**（见配图），当看到 `unhealthy` 时依次检查：路径、端口、网络策略、超时、启动延迟，然后再怀疑服务真死了。
- **为 Gateway 自身配置被动监控**：确保 Gateway 自己的 `/healthz` 也被外部监控纳管，因为如果 Gateway 自己因为文件描述符耗尽而僵死，上面的健康检查日志也就停更了，这反而是最危险的隐蔽故障。

## 总结

OpenClaw Gateway 的健康检查日志就像机器的“体温计”——读对数字才能判断发烧，强行塞退烧药只会掩盖问题。学会利用 `component` 过滤、解读 `reason` 字段、验证路径与端口，就可以把最常见的“一看日志就想重启”的冲动，转化为两次精确的配置调整。下次再看到满屏的 `unhealthy`，不妨先按这张自查表走一遍，大概率会发现代码没病，是配置病了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/0934053f9a0e9221.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/29a593e5ad22d5d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/f10bb0743ebdd12c.png)

