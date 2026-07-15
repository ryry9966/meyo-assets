---
title: OpenClaw Gateway 健康检查日志怎么看：从定位到告警的工程化拆解
feedId: 29187
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在 OpenClaw 代理网关的日常运维中，健康检查（health check）是保障上游 Agent 和 MCP 服务可观测性的基础机制。网关会按配置周期对下游的模型后端、工具插件、甚至自定义 Agent 端点发起探活请求，结果以日志形式输出。但团队常遇到两类问题：一是日志量大、刷屏严重，二是“探活失败”的语义不清，误报频繁。这篇文章会拆解健康检查日志的全链路，给出可复用的定位、解析和治理方法。

## 问题描述

通过 `docker logs` 或 `journalctl` 查看 OpenClaw Gateway 时，常常看到类似输出：

```
[healthcheck] GET /v1/agents/weather-agent/health -> 200 (2ms)
[healthcheck] GET /v1/plugins/search-mcp/health -> 503 (timeout)
```

然而，这条 503 日志可能不代表服务真正不可用——可能是超时阈值过小、路由未就绪，或上游返回了自定义状态码。并且，健康检查日志默认打在同一输出流，与业务请求日志混杂，排查时需要频繁 grep，缺少结构化的字段提取。更关键的是，网关自身对失败的重试、标记摘除等逻辑，往往在日志中体现得不够直观。

## 做法/步骤

### 1. 确认健康检查日志的生成位置与格式

在 OpenClaw Gateway 的配置（通常是 `openclaw-gateway.yaml`）中，找到 `health` 段：

```yaml
health:
  interval: 15s
  timeout: 3s
  path: "/health"
  endpoints:
    - name: agent-weather
      url: "http://weather-agent:8080/health"
    - name: mcp-search
      url: "http://search-mcp:9090/health"
  log:
    level: info
    format: structured  # 或 legacy
```

如果 `format` 设为 `structured`，日志会输出为 JSON，可直接被日志采集器解析。如果沿用旧的 `legacy` 模式，则生成纯文本行，需要后续规则提取。建议在部署时开启结构化日志。

### 2. 用 jq 或 logcli 快速过滤与统计

假设已采集到 Loki 或保留在文件，JSON 单行形式类似：

```json
{
  "ts": "2026-01-09T08:23:01.123Z",
  "level": "info",
  "component": "healthcheck",
  "endpoint": "agent-weather",
  "status_code": 200,
  "latency_ms": 2,
  "result": "pass"
}
```

常用操作：

- 查看近 5 分钟有失败的端点：
  ```
  cat gateway.log | jq 'select(.component=="healthcheck" and .result=="fail")'
  ```
- 按端点统计失败次数：
  ```
  cat gateway.log | jq -r 'select(.component=="healthcheck") | "\(.endpoint) \(.result)"' | sort | uniq -c
  ```
- 针对 Loki 的 LogQL：
  ```
  {component="healthcheck", result="fail"} | rate(5m)
  ```

如果还在用 legacy 文本日志，可以通过 awk 或 grok 模式解析，比如：
```
\[healthcheck\] (?<method>\w+) (?<url>\S+) -> (?<status>\d{3}) \((?<latency>\d+)ms\)
```

### 3. 建立健康检查日志与网关摘除逻辑的对照

健康检查失败不是终点，网关通常会对连续失败多次的端点触发摘除（unhealthy -> drain）。这部分状态变化会有单独的事件日志，关键字为 `circuit_breaker` 或 `endpoint_status_change`。排查时，需要对照两条时间线：

- 健康检查日志显示某端点开始返回 503 的时间。
- 网关标记端点 `unhealthy` 的时间（通常延迟几秒到十几秒）。
- 自动恢复后重新加入的时间。

若发现摘除动作远快于探活失败频率，很可能 `threshold` 设置过小。根据实际抖动情况，建议将连续失败阈值设为 3～5 次，间隔 15～30 秒。

## 踩坑点

**坑 1：健康检查超时与上游服务优雅停机的时差**  
当 MCP 插件滚动更新时，旧实例先进入 `SIGTERM`，但可能仍短暂接受请求并返回 503。若健康检查的超时（timeout）小于上游等待时间，网关会立即标记失败，导致流量提前切断。解决办法是将健康检查的超时时间设置为略大于上游 `terminationGracePeriodSeconds`，或者让上游先摘除自己再退出。

**坑 2：结构化日志未开启，排查成本陡增**  
很多初始部署直接使用默认的 legacy 格式，当需要对接 Loki 或 ELK 时才发现字段缺失。建议在初始部署模版中就启用 `structured` 格式，并统一字段名。

**坑 3：探活路径与实际业务接口不匹配**  
有些自定义 Agent 服务仅提供一个简单的 `/health` 回显，但真正的处理端口可能因内存不足而 hang 住。健康检查通过不代表业务可用。此时需要将探活路径指向一个轻量的“自检”接口（如 `/healthz` 并内部检查队列深度），并在日志中区分 `liveness` 与 `readiness`。

**坑 4：健康检查日志的采样与丢弃失度**  
全量记录每个检查周期会产生大量日志，但若全量关闭，出问题时又无数据回溯。建议利用 OpenClaw Gateway 的 `log.sample_rate` 参数，只记录失败、状态变更的探活结果，或按 1/10 比例采样成功日志。同时，失败日志必须 100% 输出。

## 可复用建议

- **日志管道化**：不管是 filebeat + Kafka 还是 promtail + Loki，提前规划好结构化日志的 schema。健康检查日志的字段至少包含：`timestamp`, `endpoint_name`, `result`, `status_code`, `latency_ms`, `error_message`。
- **告警分级**：不要对单次 503 直接告警。设置规则：连续 2 分钟或 3 次失败才触发 warning；5 次及以上触发 critical。并联动网关摘除事件，避免重复呼叫。
- **面板构建**：用 Grafana 建两个面板：一个显示各端点健康检查成功率的时序图，一个表格列出当前处于 unhealthy 状态的端点及其最后失败原因。这样能快速从“日志查”转为“看板判别”。
- **自定义探活逻辑**：如果你的 Agent 或 MCP 服务需要更精细的检测，可以开发一个 sidecar 探活器，输出 OpenClaw 兼容的健康检查端点，然后在 gateway 配置中指向这个 sidecar。日志仍然统一。

## 总结

OpenClaw Gateway 的健康检查日志是观测代理层与下游服务连接状态的“第一现场”。理顺日志格式、建立结构化过滤、贯彻告警分级，能把“看日志”从低效滚动变为精准定位。关键是在部署阶段就启用结构化输出，且让健康检查的超时、阈值与上游行为对齐。剩下的，就是养成先查面板、再下钻日志的习惯，减少误判。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/70b64f6d8c60577e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/5aeb8aa6622f7bac.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/edfb8462d00ba095.png)

