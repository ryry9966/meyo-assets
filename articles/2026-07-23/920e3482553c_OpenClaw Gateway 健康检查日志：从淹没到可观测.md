---
title: OpenClaw Gateway 健康检查日志：从淹没到可观测
feedId: 30137
source: 综合讨论
publishedAt: 2026-07-23
---

# OpenClaw Gateway 健康检查日志：从淹没到可观测

## 背景

在基于 OpenClaw Gateway 搭建的 Agent / MCP 服务网格中，健康检查是服务发现、负载均衡和熔断策略运转的基石。网关不仅为上游 MCP 服务提供路由，自身也会暴露 `/health` 或类似的端点给 K8s、负载均衡器或外部监控系统。一旦出现间歇性路由异常或上游被意外摘除，第一反应往往是翻看健康检查日志。

但默认配置下，健康检查日志与业务访问日志混杂在同一个输出流中。如果探测间隔短至数秒，几分钟就能产生成千上万条日志——排查问题如同大海捞针。

## 问题

以下场景不少社区用户都遇到过：
- `access.log` 被每秒一次的 K8s liveness probe 撑爆，磁盘 I/O 激增；
- 想快速统计某个上游的健康检查失败率，但日志里缺乏结构化字段，只能靠复杂的文本处理；
- 不确定某次 503 是网关自身不健康，还是上游探测超时。

这些痛点归根结底在于**健康检查日志缺乏独立处理与可读性**。

## 做法 / 步骤

### 1. 启用结构化日志并确定关键字段

OpenClaw Gateway 支持 JSON 格式的访问日志。典型配置：

```yaml
access_log:
  path: /var/log/openclaw/access.log
  format: json
  fields:
    - request_time
    - status
    - upstream_addr
    - upstream_status
    - request_path
    - remote_addr
```

如果历史日志是文本格式，建议尽快迁移到 JSON，方便后续流程化处理。

### 2. 快速过滤健康检查请求

假定健康检查路径为 `/healthz`，用 `jq` 筛选：

```bash
cat access.log | jq 'select(.request_path == "/healthz")' > health_check.log
```

实时监控场景下可直接管道：

```bash
tail -f access.log | jq --unbuffered 'select(.request_path == "/healthz")'
```

若要进一步区分网关自身探针与上游探针，可借助 `remote_addr` 或自定义 Header（如 `X-Health-Check-Type`）。

### 3. 聚合分析，暴露异常

按上游地址统计平均响应时间和非 200 状态码占比：

```bash
jq -s '
  group_by(.upstream_addr) |
  map({
    upstream: .[0].upstream_addr,
    avg_time: (map(.request_time) | add / length),
    failure_rate: (map(select(.upstream_status != 200)) | length / length)
  })
' health_check.log
```

一旦某上游的 `failure_rate` 超过阈值，可以先行触发告警，无需等待业务方反馈。

### 4. 分流日志输出

OpenClaw Gateway 允许定义多个 `access_log` 并配合条件变量。先定义一个变量判断是否为健康检查：

```nginx
map $request_path $is_health_check {
    default          0;
    /healthz          1;
    /readyz           1;
}
```

随后单独输出健康检查日志：

```nginx
access_log /var/log/openclaw/health.log json if=$is_health_check;
access_log /var/log/openclaw/access.log json if=$is_health_check!=1;
```

这样主日志立即瘦身，健康检查日志可独立配置 rotation 策略（如保留 1 天）。

### 5. 日志轮转与监控

健康检查日志增长快且利用价值有限，建议使用 `logrotate` 设置较短留存：

```
/var/log/openclaw/health.log {
    daily
    rotate 1
    compress
    missingok
    notifempty
}
```

同时，在监控侧对健康检查日志中 `status >= 500` 的条数进行采样告警，而不是等到业务日志报错。

## 踩坑点

- **高频率探测淹没主日志**  
  如果未做分流，K8s 探针每 1 秒产生 2 条日志（`/healthz` + `/readyz`），一天就是 172,800 条，几乎将所有定位线索冲淡。

- **误将上游健康检查超时混为网关日志**  
  网关在主动探测上游时若超时，可能会记录一条 error 日志，排查时需注意区分 `upstream_addr` 是否为自己，避免得出“网关自身不健康”的错误结论。

- **日志级别混杂 debug 信息**  
  问题排查时临时开启 debug 级别，记得事后恢复。否则每条健康检查都可能附带冗长堆栈，磁盘压力和解析效率都会明显恶化。

- **忽视代理层的心跳**  
  网关前端如果有云负载均衡器，它会持续向网关发送 health check，且来源 IP 固定。可利用 `remote_addr` 排除或标记此类流量，避免干扰对真实异常的判断。

## 可复用建议

1. **结构化日志为底线**  
   不要使用空格分隔的文本日志，字段一旦增减，解析脚本全部失效。JSON 字段可随版本演进向下兼容。

2. **分离并短留健康检查日志**  
   将健康检查日志独立输出，保留 1～2 天即可，同时为主日志节省 90% 以上的磁盘 I/O。

3. **规范健康检查路径与 Header**  
   统一使用 `/healthz`、`/readyz`，并在请求中注入 `X-Health-Check-Source: k8s-probe` 等自定义头，便于分析和分流。

4. **建立基于日志的探活监控**  
   利用 `jq` 等工具每 30 秒分析一次健康日志中非正常响应的比率，对敏感上游设置 < 1% 失败率的告警规则，实现比单纯看心跳更精确的可用性观测。

## 总结

健康检查日志表面上是“心跳噪声”，实际是服务网格可观测性最前哨的一环。通过结构化、分流、短留存和聚合监控四个动作，你可以把淹没性的日志流调教成清晰的指示灯。下次再遇到路由抖动，直接打开 `health.log`，用一条 `jq` 命令就能定位是哪个上游在频繁“假死”——这比翻千百行业务日志要务实的多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/fa3a06888c98053c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/321e2e8ad6ab45e7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/cf9c3b0982006ce7.png)

