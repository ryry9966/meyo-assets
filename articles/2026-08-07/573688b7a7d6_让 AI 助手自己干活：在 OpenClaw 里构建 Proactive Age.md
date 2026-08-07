---
title: 让 AI 助手自己干活：在 OpenClaw 里构建 Proactive Agent 的工程实践
feedId: 31948
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

当前多数 AI 助手都工作在“请求-应答”模式下：用户提问，模型回复。即便已经接入了 MCP 工具和外部 API，触发起点仍然是用户的一句指令。但在运维巡检、数据监控、信息聚合等场景中，真正有价值的往往是 **proactive 能力**——不等你开口，模型就可以根据预设条件主动执行信息收集、判断并采取行动。

OpenClaw 社区中，Agent 已经具备工具调用（MCP）、长短期记忆和外部动作的能力。唯一的缺口就是 **触发机制**。本文基于实际搭建的一个 proactive 助理，记录如何把定时调度、条件监控和 Agent 决策串联起来，形成一套可控、可观测的自动化工作流。

## 问题定义

我们需要实现一个 **服务器健康巡检助理**，每天 09:00 自动执行以下逻辑：

1. 通过 MCP 工具查询目标服务器的 CPU/内存/磁盘指标（Prometheus 或自定义 HTTP 接口）。
2. 判断是否有指标超过阈值，若有则进一步查询最近 1 小时的日志。
3. 综合评估后，若需人工介入则发送 Slack 通知，否则记录一份简要的日报到 Git 仓库。

整个过程中，用户无需发送任何消息，Agent 主动完成从数据获取到决策输出的闭环。同时需要保证：

- 行动可审计，每次决策都有日志。
- 关键通知必须包含上下文，避免“狼来了”式告警。
- 工具调用失败时有退避策略，不丢失巡检任务。

## 实现步骤

### 1. 为 Agent 准备 MCP 工具

我们需要三个工具：

- `query_metrics(target, metric_type)`：查询服务器指标，返回数值。
- `fetch_logs(target, timerange)`：拉取指定时间段的 systemlog 片段。
- `send_slack(channel, message)`：推送通知。

这部分可以用现成的 MCP server 包装，例如使用 [mcp-server-http](https://github.com/anthropics/mcp) 暴露内部运维 API，或者直接用 Python 写一个简单的 stdio MCP 服务器挂在 OpenClaw 下。这里给出简化版 `mcp_server_config.yaml` 片段：

```yaml
mcp_servers:
  ops-tools:
    command: python
    args: ["-m", "ops_mcp_server"]
    env:
      PROMETHEUS_URL: "http://prom.internal:9090"
      SLACK_WEBHOOK: "https://hooks.slack.com/..."
```

MCP 服务器的具体实现不是本文重点，但需要注意所有外部调用都必须设置合理的超时（不超过 10 秒），避免阻塞 Agent 推理循环。

### 2. 设计触发调度器

触发层与 Agent 完全解耦。使用 cron 或 systemd timer 在服务器上定期运行一个轻量级 Python 脚本，该脚本的唯一任务是调用 OpenClaw 的 API，向指定的 Agent 发送一条“隐形触发”消息。这样做的好处是保持 Agent 的无状态，任何外部调度器都可以接入。

示例脚本 `trigger_health_check.py`：

```python
import requests
import os

OPENCLW_API = os.environ["OPENCLW_API_URL"]
AGENT_ID = "proactive-ops-agent"
TRIGGER_MSG = "system:trigger:daily_health_check"

response = requests.post(
    f"{OPENCLW_API}/agents/{AGENT_ID}/messages",
    json={"content": TRIGGER_MSG, "metadata": {"source": "cron"}},
    timeout=30
)
response.raise_for_status()
```

`crontab` 条目：

```
0 9 * * * /usr/local/bin/python /opt/scripts/trigger_health_check.py
```

Agent 在 system prompt 中预设了处理该隐形指令的逻辑，不会对用户可见。

### 3. 编写 Agent 的 System Prompt

这是 proactive 行为的核心。我们需要在 system prompt 中明确告诉模型：

- 当收到以 `system:trigger:` 开头的消息时，这是一个后台任务，不需要等待用户后续输入，应立即开始执行预定义流程。
- 必须使用工具收集数据，先查询指标，再根据阈值决定是否拉日志。
- 每个步骤的输出要简洁结构化，最终动作只能是“记录日报”或“记录日报+发 Slack”。
- 严格禁止幻觉数据，所有数据必须来自工具返回。

Prompt 模板（节选）：

```
You are a proactive infrastructure assistant.
When you receive a message starting with "system:trigger:daily_health_check", execute the following routine immediately:

1. Call query_metrics for each server in the monitoring list (fetched from an environment variable or a configuration tool).
2. If any metric exceeds threshold (CPU>80%, MEM>90%, disk>85%), call fetch_logs for that server with a one-hour window.
3. Summarize the status and any anomalies found. Prepare a daily report in Markdown format.
4. If anomalies exist, use send_slack to notify #ops-daily. Otherwise, only append the report to the daily log file via another MCP tool (e.g., git_commit_note).
5. After completion, output a concise summary indicating "Done: daily_check" and the number of anomalies.
```

### 4. 配置 Agent 与放置监控列表

在 OpenClaw 中创建一个新 Agent，关联上述 MCP 服务器，并将 system prompt 写入。另外，我们可以将监控列表写在一个 MCP 资源中（或直接放在 `env` 里），例如通过 `config` 工具返回 `["web-01","web-02","db-01"]`。Agent 在收到触发指令后会自动调用该工具获取目标列表，避免硬编码。

至此，proactive 助理就搭建完成了。实际运行时，Agent 会在每天 9 点自动开始巡检，完全不需要人工干预。

## 踩坑记录

### 坑 1：MCP 工具超时导致 Agent 卡死

早期未在 MCP 服务器端设置超时，某次 Prometheus 响应过慢（>30 秒），导致 Agent 等待工具返回时占满处理窗口，后续触发消息被阻塞。**解决**：在 MCP 客户端（OpenClaw 侧）设定工具调用超时 15 秒，并在工具实现里加入 `signal.alarm` 或 `asyncio.wait_for`，超时后返回标准化错误，让 Agent 能优雅跳过并记录异常，而不是彻底停止。

### 坑 2：重复触发与幂等性问题

如果 cron 配置错误或手动测试，可能会在同一分钟触发多次。由于 Agent 没有内置防重机制，产生了重复的 Slack 通知。**解决**：在触发脚本中引入基于 Redis 的简单锁（`SET NX EX`），任务开始前获取锁，执行完成后释放。锁的 key 使用当天日期，确保每天只运行一次。

### 坑 3：Agent 幻觉数据

某次工具返回格式变化，Agent 直接编造了一个看似合理的 CPU 值，导致日报失真。**解决**：在 system prompt 中显式约束：若工具返回 `error` 或无法解析，必须标记该项为 `unknown` 并跳过后续分析，不得填充默认值。同时在发送报告前增加自检步骤：要求 Agent 列出所有数据来源及原始返回值。

### 坑 4：权限过度

最初在 MCP 工具里直接给了 `kubectl scale` 权限，打算实现自动扩容。测试时差点把生产环境副本数调到 0。**教训**：Proactive 行动必须遵循最小权限原则，高风险操作只允许在人工确认后才能执行。我们的最终方案中，Agent 只能查询和通知，所有写操作都被移除，改为生成工单。

## 可复用建议

1. **将触发、决策、执行三层分离**。调度器只负责触发，Agent 只负责决策与轻量执行（读操作+通知），危险操作交给独立的 action runner（需审批）。这样即使模型幻觉，影响也被限制。
2. **利用 MCP 的订阅机制增强实时性**。如果 MCP 服务器支持 resources 的订阅/通知（WebSocket 或 SSE），可以用事件驱动代替定时轮询，例如监控 GitHub issue 创建就主动分类。OpenClaw 的 Agent 能直接通过 MCP notifications 接收事件，从而构建完全 event-driven 的 proactive 能力。
3. **统一任务日志**。将所有 proactive 任务的状态、决策摘要、耗时写入结构化日志（例如 JSON lines），便于后续审计和调试。我们使用 Agent 调用的最后一个工具 `log_task` 来完成这一步，确保人类可追踪。
4. **小范围灰度**。先在预发环境运行一周，观察 Agent 的行为模式和幻觉概率，再推广到生产级通知。可以用 `dry_run` 模式让 Agent 只记录不发送。

## 总结

在 OpenClaw 的 Agent + MCP 架构下，实现 proactive 能力的核心不是改造模型本身，而是通过 **外部调度 + 结构化 prompt + 工具边界控制** 把 AI 嵌入到定时工作流里。虽然过程中会遇到超时、重复触发、幻觉等实际问题，但只要将权限收紧、执行过程可观测，这种模式就能显著提升运维效率，让 AI 从“问答机”进化为真正的自动化协作者。

同样的思路也可以推广到代码仓库事件响应、市场数据定时分析等场景，区别只在于触发源和工具集的替换。希望这篇工程笔记能帮你少踩一些坑，让 AI 真的自己动起来。

---

