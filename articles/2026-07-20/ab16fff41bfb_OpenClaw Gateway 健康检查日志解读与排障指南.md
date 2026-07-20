---
title: OpenClaw Gateway 健康检查日志解读与排障指南
feedId: 29799
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景

在 OpenClaw 的部署拓扑里，Gateway 是所有 Agent 与 MCP 工具链的入口。无论你走的是子域路由、路径转发，还是直连 IP，Gateway 的健康检查 （health check） 都是决定上游负载均衡、自动摘流、甚至告警触发的关键依据。

但多数人拿到健康检查日志的第一反应是：**信息太多、字段陌生、看不出到底健不健康。** 这篇帖子从一线运维视角出发，梳理 `openclaw-gateway` 健康检查日志的结构、常见状态码含义、快速排障路径，以及几个容易被忽略的“假健康”风险点。

---

## 健康检查日志的位置与格式

默认情况下，健康检查请求由 Gateway 自身发起（对内服务端口）或由前置代理/负载均衡器发起（对外入口）。对应的日志位置取决于你的启动方式：

- **systemd 部署**：`journalctl -u openclaw-gateway -f | grep health`
- **容器化部署**：`docker logs -f openclaw-gateway 2>&1 | grep "health_check"`
- **裸进程**：查看 `logs/gateway.log` 或 `--log-file` 指定的路径

典型一条健康检查日志长这样：

```
2025-01-15T10:23:45.621Z  INFO health_check{method=GET path=/healthz status=200 latency=2ms remote=10.0.1.12} => ok
```

字段虽少，但隐含信息足够判断大部分问题。核心关注点：

| 字段 | 含义 | 异常表现 |
|------|------|----------|
| `status` | HTTP 响应码 | 非 200/204 即为不健康 |
| `latency` | 本次检查耗时 | 突增可能意味着依赖组件慢响应 |
| `remote` | 发起检查的客户端 IP | 用于区分内外部检查源 |
| `path` | 检查路径 | `/healthz` 是默认就绪探针，`/ready` 用于流量接入 |

**特别提醒**：如果你看到 `path=/health`，那通常是旧版配置残留，务必确认探活路径与负载均衡器配置一致，否则会出现“LB 认为 down，但 Gateway 自己觉得 up”的割裂。

---

## 做法 / 步骤

### 1. 确认到底谁是检查方

先搞清楚日志里的 `remote` 是谁：

- 如果 IP 是 `127.0.0.1` 或 Pod IP，大概率是本地 kubelet / 前置 sidecar；
- 如果是其他私有 IP，多半来自内部 LB 或服务网格的 health checker；
- 如果是公网 IP —— 你最好检查一下防火墙规则，不该把健康接口暴露出去。

**踩坑点**：某些云厂商的负载均衡器会在健康检查包里夹带自定义 Header，Gateway 如果没有放行这些 Header，可能返回 400。检查日志里 `status=400` 且 `latency<1ms`，基本就是请求被直接在协议层拒绝，先看 `allowed_health_check_headers` 配置。

### 2. 理解“健康”的判定逻辑

OpenClaw Gateway 的健康端点内部做了三层检查（按版本 v0.9+）：

- **L1 进程存活**：单纯返回 200，这个永远不应该失败，除非进程 hang 住。
- **L2 上游依赖就绪**（仅 `/ready`）：检查 MCP Server 连接池、认证服务可达性。日志中如果出现 `dependency=mcp_pool status=degraded`，意味着部分 MCP 后端连不上，但 Gateway 本身仍可能判为健康（取决于 `degraded_as_healthy` 配置）。
- **L3 业务逻辑探针**（可选）：由 Lua 脚本或 Wasm 插件自定义，日志里会有 `probe=custom` 标记。

**常见误判**：看到 `/healthz` 返回 200 就认为服务正常，实际 `/ready` 可能因为线程池耗尽而超时。**永远让负载均衡器指到 `/ready` 路径，除非你只关心进程是否存活。**

### 3. 逐类排查异常日志模式

根据日志状态码快速定位：

- **连续 `status=503`**：Gateway 正在优雅关闭，或主动摘流。检查 `pre_stop_delay` 是否足够让 LB 先摘除。
- **间歇 `status=502` + latency 升高**：上游 MCP Server 响应慢导致 Gateway 判定自身不可用。去 Gateway 配置里调大 `health_check_timeout` 和 `upstream_connect_timeout`。
- **`status=404`**：路径写错，90% 是负载均衡器配置页里路径填成 `/health` 或 `/healthcheck`。

一个真实场景：Gateway 滚动更新时，旧 Pod 的 `/ready` 在收到 SIGTERM 后立即变成 503，但 LB 的健康检查间隔是 5 秒，导致 3~4 秒的流量 blackhole。**对策**：设置 `delay_before_shutdown=10s`，保证日志里出现 `shutdown_delayed` 记录，而不会直接跳到 503。

---

## 踩坑点

1. **日志级别过低导致关键信号被淹没**  
   健康检查日志默认 INFO 级，如果日志冲击过大，有人会改成 WARN，结果连探活失败都不再打印。建议保持 INFO，但通过 `rate_limit` 限制健康检查日志条数（例：`health_log_max_per_sec=5`）。

2. **IPv6 与 IPv4 双栈导致 remote 字段混淆**  
   主机的 localhost 检查可能以 `::1` 出现，而 LB 以 IPv4 请求，你以为有多个检查源，误判为异常流量。

3. **TLS 卸载后不匹配协议**  
   如果 LB 与 Gateway 之间是 HTTP，但 Gateway 强制要求 HTTPS（`force_https=true`），健康检查请求会收到 307 重定向，LB 可能把它当失败。日志里会看到 `status=307`，需关闭该 listener 的 HTTPS 强制。

4. **日志里的 `probe=custom` 与重试风暴**  
   自定义探针脚本若执行时间过长，会导致多个 LB 检查请求堆积，Gateway 线程池被占满，触发雪崩。日志中会先出现大量 200，然后突然全部超时。务必给自定义探针添加超时控制。

---

## 可复用建议

- **为健康检查日志建独立流**：在 log config 里把 `health_check` 事件输出到单独的滚动文件，避免被业务日志干扰。
- **关键指标告警**：将 `health_check_failure_rate` 和 `ready_check_failure_rate` 接入 Prometheus，阈值设为连续 2 次失败告警，而非单次。
- **模拟探活脚本**：本地用 `watch -n 2 curl -v http://127.0.0.1:9020/ready` 持续采样，与日志对照理解状态切换时机。
- **区分探活与就绪的监控图**：在 Grafana 里用两个面板分别展示 `/healthz` 和 `/ready` 的响应码分布，避免混淆。

---

## 总结

OpenClaw Gateway 健康检查日志不是黑盒。只要抓住 `status`、`path`、`remote` 三个字段，就能还原出这场 “自证清白” 的过程。排障技巧本质上是**把日志当时间轴，还原检查方、Gateway、后端依赖三者的对话**。遇到“明明活着却被摘流”的问题时，先从日志里确认是不是 `/ready` 路径返回了非 200，再去深挖依赖超时或优雅关闭的时序。

最后一条忠告：**永远不要忽视一条 2ms 的 200 日志背后 5 分钟的配置偏差。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/fe94999a30f15f20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/b6bf9a33d5fd3721.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/be9245ed13ec35c5.png)

