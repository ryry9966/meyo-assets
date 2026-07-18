---
title: OpenClaw Gateway 健康检查日志解读与排障实践
feedId: 29583
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

OpenClaw Gateway 是 Agent 与 MCP 插件体系中的 API 流量入口，负责路由、认证、限流和协议转换。在生产集群中，Kubernetes 或 Docker 调度器会定期向 Gateway 发起健康检查（Liveness / Readiness Probe），一旦连续失败就会触发容器重启或摘除流量。健康检查日志因此成为服务可用性的第一手信号。但很多团队对 `/healthz` 返回 200 就认为万事大吉，实际上“假健康”导致的故障并不少见——探针返回成功，但真实请求却大量 502，问题就藏在那些看似正常的健康检查日志里。

## 常见问题

- 健康检查突然失败，但服务进程并未退出，Pod 被反复重启
- 监控显示健康检查延迟抖动，但又不算超时，告警时有时无
- 健康检查日志刷屏，淹没了真正的业务错误
- 明明上游 Agent 服务挂了，Gateway 的健康检查仍然返回 OK

这些问题都指向一个核心能力：读懂健康检查日志，并据此配置合理的检查策略。

## 做法 / 步骤

### 1. 明确健康检查端点及其逻辑

OpenClaw Gateway 默认暴露两个端点（具体路径可能因部署配置而异）：

- `/healthz/live`：Liveness probe，仅检查 Gateway 自身进程是否存活，通常不检测外部依赖。
- `/healthz/ready`：Readiness probe，用于决定流量是否接入。会逐一检查关键依赖，例如 Agent 服务、MCP 后端、Redis 会话存储、数据库等。

通过查看 Gateway 配置文件或启动参数，确认健康检查的依赖列表和超时设置。例如：

```yaml
health:
  endpoints:
    liveness: "/healthz/live"
    readiness: "/healthz/ready"
  checks:
    - name: "agent-upstream"
      type: "http"
      url: "http://agent-service:8080/health"
      timeout: 2s
    - name: "redis"
      type: "redis"
      dsn: "redis:6379"
      timeout: 1s
```

### 2. 打开详细日志

默认日志级别通常为 `info`，健康检查成功时只输出一行状态码，看不到内部检查过程。要定位间歇性失败或延迟抖动，需要临时调高日志级别：

```bash
export LOG_LEVEL=debug   # 或者通过启动参数 --log-level debug
```

重启 Gateway 后，观察日志。每一次探针请求都会输出完整的检查过程，例如：

```
time=2025-03-14T10:06:00Z level=debug msg="health check started" endpoint=/healthz/ready
time=2025-03-14T10:06:00Z level=debug msg="checking component" component=agent-upstream
time=2025-03-14T10:06:00Z level=debug msg="check passed" component=agent-upstream latency=12ms status=200
time=2025-03-14T10:06:01Z level=debug msg="checking component" component=redis
time=2025-03-14T10:06:01Z level=debug msg="check passed" component=redis latency=3ms
time=2025-03-14T10:06:01Z level=debug msg="health check completed" endpoint=/healthz/ready status=200 total_latency=15ms
```

如果某个组件失败，日志会明确给出原因：

```
level=error msg="check failed" component=agent-upstream error="context deadline exceeded" latency=2001ms
```

此时整个 ready 端点会返回 503，Liveness 仍然保持 200。

### 3. 分析日志中的关键信号

- **延迟异常**：某组件检查耗时长，说明该依赖响应变慢，需要尽快扩容或排查网络。
- **间歇性超时**：若日志出现 `context deadline exceeded` 且延迟恰好在超时边界，说明探针超时设置过短或依赖存在波动的慢查询，需调整 `timeout` 参数。
- **只有 Liveness 失败**：极少见，通常意味着 Gateway 自身 panic 或者被 OOM Kill。日志会记录 panic 堆栈。
- **Readiness 失败但业务请求正常**：大概率是 readiness 里配置了非关键依赖，或者依赖的检查 URL 有问题，应剔除不必要的检查项。

### 4. 合理控制日志输出量

Kubernetes 的探针默认间隔 10 秒，三副本集群每分钟会产生 18 条 debug 日志。如果开了 debug 级别，日志体积会快速增长。建议：

- **常态化**：生产环境将日志级别恢复为 `info`。仅在排查问题时临时开启 debug。
- **条件日志**：如果 Gateway 框架支持，可以配置仅当健康检查失败时才输出详细原因，成功时静默。例如 Logrus 的 `log_sampling` 或自定义 Hook。
- **分离日志流**：将 health check 日志单独输出到文件，通过 `access_log off` 或类似指令禁用探针的 access log，避免和业务日志混杂。

## 踩坑点

1. **Readiness 包含了过多外部依赖**  
   曾经一个团队把消息队列、对象存储、甚至第三方 API 都塞进 Readiness 检查。结果第三方间歇性超时导致整个 Gateway 频繁摘除流量，引发雪崩。原则：Readiness 只检查**强依赖**，且每个依赖都必须有合理的超时和重试。

2. **探针超时小于依赖检查总耗时**  
   Kubernetes 的 `timeoutSeconds` 与 Gateway 内部的 `timeout` 是两层控制。如果 pod spec 中 readiness probe timeoutSeconds=3，而 Gateway 配置的总检查超时为 5 秒，探针会在 3 秒时直接 kill 连接，日志里只会看到客户端断开，看不到真正的错误信息。一定要让 K8s 探针超时大于 Gateway 检查总耗时至少 1 秒。

3. **健康检查日志未结构化**  
   纯文本日志很难自动化告警。确保日志包含 `component`、`latency`、`error` 等字段，并接入 Loki / ELK。创建 “Readiness 检查失败” 和 “依赖组件延迟 > 500ms” 的告警规则，比只看整体 503 更前置。

4. **Liveness 与 Readiness 的执行体相同**  
   有时配置错误导致两个端点执行完全相同的检查，Liveness 探针因依赖超时而杀死进程，加剧系统不稳定。应严格区分：Liveness 只返回进程是否存活，不做任何外部调用。

## 可复用建议

- 在 Gateway 启动时打印一次健康检查配置摘要到日志，包括检查的依赖列表和超时值，方便事后回溯。
- 为每个依赖的健康检查添加 Prometheus metrics（`gateway_health_check_duration_seconds`、`gateway_health_check_failures_total`），通过 Grafana 面板直观展示。
- 当出现健康检查失败时，第一时间比较同一时间点 Gateway 的日志与依赖服务的日志，确认是网络、资源还是逻辑故障。
- 将健康检查端点保护起来，限制只能由调度器 IP 访问，避免被外部扫描或无意义的探测日志骚扰。

## 总结

OpenClaw Gateway 的健康检查日志并不是简单的“成功/失败”二进制信号，它是服务和依赖链路的微观测温计。通过打开 debug 日志、关注组件级检查结果、并合理控制输出与探针配置，可以将很多线上隐患消灭在流量受损之前。记住一句话：只要 Readiness 日志里的任何一个依赖在呻吟，就代表整个 Gateway 离 502 风暴不远了。把这些日志用好，比加十个监控告警更直接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/42db899b7339a60e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/270dbf68a9328411.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/5cccbf8a34611d2f.png)

