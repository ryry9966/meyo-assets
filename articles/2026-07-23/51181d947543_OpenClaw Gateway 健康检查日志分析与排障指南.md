---
title: OpenClaw Gateway 健康检查日志分析与排障指南
feedId: 30123
source: 综合讨论
publishedAt: 2026-07-23
---

# OpenClaw Gateway 健康检查日志怎么看

## 背景：为什么健康检查日志值得专门读

在生产环境里，OpenClaw Gateway 作为流量入口，承担着向后端服务（Agent、MCP 工具、插件 Runtime）转发请求的职责。一旦上游节点出现延迟升高、连接拒绝或间歇性 5xx，网关会通过健康检查（health check）机制自动将其标记为不可用，从负载均衡池中剔除。

这些动态上下线的行为全部记录在健康检查日志里。很多团队直到上游服务的告警响起，才去翻日志，却发现被大量“probe ok”淹没，关键的状态切换信息反而被忽略了。读懂这些日志，可以在用户感知到异常之前就发现问题。

## 常见痛点

- **日志量过大**：主动健康检查周期短（如 5 秒一次），成功记录刷屏，难以定位异常。
- **被动与主动混淆**：被动检查（基于真实请求的失败）和主动检查（定时探活）的日志混杂，关键字不一致，排查费力。
- **状态变化不直观**：只知道当前是 UP 或 DOWN，不看到完整的状态转换历史，溯源根因困难。

## 第一步：找到并识别健康检查日志

OpenClaw Gateway 的健康检查日志通常与 error_log 或 access_log 集成，但也有独立分类。确认你的 `gateway.yml` 或插件配置中是否设置了：

```yaml
plugins:
  health_check:
    log_level: info          # 建议调整为 debug 以获得详细失败原因
    log_file: /var/log/openclaw/hc.log   # 独立文件，避免干扰
```

如果不指定独立文件，日志会写入主错误日志，行首可能带有 `[healthcheck]` 前缀。典型的主动健康检查记录长这样：

```
2025-03-11T14:22:01.233+08:00 [healthcheck] probe upstream=agent-worker-1 addr=10.2.1.15:8080 type=http status=200 latency=0.003s result=success
```

被动检查触发时，会在代理请求日志中附加类似标记：

```
... [healthcheck] passive event: upstream=agent-worker-2 status=502 failures=3/5 threshold_reached=1 action=mark_down
```

这两类日志的含义不同，排查时务必区分。

## 第二步：解析关键字段

想高效排障，必须明确每个字段的工程含义：

- **`type`**：`http` / `tcp` / `grpc`。不同协议的失败处理逻辑不同，GRPC 的健康检查会依赖标准 health check service，超时门槛更敏感。
- **`status` / `result`**：主动检查时 `result=success/failure`，同时给出 HTTP 状态码。被动检查只记录 `status`，比如 502/504 会触发失败计数。
- **`failures` vs `threshold`**：被动健康检查中的失败计数器。例如 `failures=3/5` 代表 5 次滑动窗口内失败了 3 次，触及阈值则标记为 DOWN。理解这个滑动窗口大小（默认 10 秒或可配）有助于判断是否因瞬时抖动下线。
- **`latency`**：探测延迟超出主动检查的 `timeout` 会直接判定失败，这个是排查超时的直接依据。
- **`action=mark_down` / `action=mark_up`**：状态切换事件，是需要重点报警的日志行。

## 第三步：实战排查——从日志到结论

**场景：某个 MCP 工具服务突然无法调用，但稍后恢复**

1. **筛选状态切换日志**  
   ```bash
   grep -E "action=mark_(up|down)" /var/log/openclaw/hc.log
   ```
   假设输出：
   ```
   2025-03-11T15:10:03 action=mark_down upstream=mcp-tool-search failures=3/3 threshold=3
   2025-03-11T15:10:18 action=mark_up   upstream=mcp-tool-search result=success
   ```
   说明该节点因 3 次连续失败被摘除，15 秒后主动检查成功自动上线。

2. **倒查失败原因**  
   紧接 `mark_down` 之前的被动或主动检查日志：
   ```
   2025-03-11T15:10:01 passive event: upstream=mcp-tool-search status=502 failures=1/3
   2025-03-11T15:10:02 passive event: upstream=mcp-tool-search status=502 failures=2/3
   2025-03-11T15:10:03 passive event: upstream=mcp-tool-search status=502 failures=3/3 threshold_reached=1 action=mark_down
   ```
   三秒内连续 502，很可能上游服务重启或负载过高。需要进一步查上游应用日志。

3. **识别“幽灵探测”**  
   如果主动检查的 `latency` 呈现周期性尖刺（如每 30 秒一次 2 秒以上），而服务本身正常，可能是 GC 停顿导致探测超时。可以调大该 upstream 的 `timeout` 或调整主动检查间隔（`interval`），避免误摘。

## 踩坑点

- **被动检查与主动检查阈值混淆**  
  默认配置下，被动检查只需 3 次失败就摘除，而主动检查通过后立即上线。如果上游在压力下时好时坏，会出现频繁 flip-flop。建议将被动检查的 `unhealthy.threshold` 和主动检查的 `healthy.up_threshold`（连续成功次数）错开配置，比如被动失败 5 次，主动恢复需连续成功 3 次。

- **日志级别不够**  
  生产环境通常只开 `warn`，可能导致关键的被动失败计数日志不输出。建议在 `health_check` 插件中显式设置 `log_level: info`，并为主动检查单独配置较低级别，避免淹没。

- **多上游时日志筛选困难**  
  当网关后端有几十个 Agent 或工具时，强烈建议在日志格式中加入 `upstream` 标签，并利用日志管理工具按 upstream_name 做索引和告警，而不是全靠 grep。

## 可复用建议

- **结构化日志改造**：将日志输出为 JSON 格式，便于采集到 Loki/ES 中做统计。例如统计每 5 分钟的 up/down 事件次数。
- **设置基于日志的告警**：当检测到 1 分钟内同一 upstream 的 `action=mark_down` 出现超过 2 次，触发告警通知。
- **保留历史状态**：将状态转换日志接入 ClickHouse 或其它时序存储，回溯一个服务的可用性历史比只看当前状态有价值得多。
- **主动检查的“预热”配置**：新节点上线后，设置 `initial_delay` 避免因服务未完全就绪导致误 mark down。

## 总结

OpenClaw Gateway 的健康检查日志不止是“线上拨测记录”，而是实时反映上游服务健康度的温度计。通过区分主动与被动检查、抓住状态切换事件、理解计数器和阈值窗口，你可以把排障时间从“翻好久日志”缩短到几秒钟。

下次当某个 Agent 服务默默挂掉又恢复时，别去翻监控大盘，先在日志里搜一次 `action=mark_down`，往往答案就在那里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/dee930df9a7df2a4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/a26ca1e13d6cd1df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/83300d96a48ec296.png)

