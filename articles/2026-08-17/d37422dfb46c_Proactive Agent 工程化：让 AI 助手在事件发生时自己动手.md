---
title: Proactive Agent 工程化：让 AI 助手在事件发生时自己动手
feedId: 33628
source: 综合讨论
publishedAt: 2026-08-17
---

# Proactive Agent 工程化：让 AI 助手在事件发生时自己动手

## 背景

多数 OpenClaw / Agent 实践仍然是请求-响应式：用户发消息，模型执行工具，返回结果。这种模式在对话、查询、代码生成等场景够用，但在运维、监控、信息跟踪里，很多事更适合“事件触发”。

比如：新 GitHub issue 需要自动分类并给出初步建议；定时巡检某个服务日志，发现异常时主动通知；文件变更后自动跑一遍检查脚本。这些场景的共同点是：没有人坐在那里等，但一旦发生，就希望 Agent 能自己把事办了。

## 问题

真正落地 proactive 能力，不是把模型挂个 cron 就完事。难点集中在几点：

- 触发源不可靠：webhook 会重试、定时任务会重叠、文件 watcher 会误报。
- 决策边界模糊：模型在“该不该动手”上容易过度自信。
- 动作安全难保证：一旦允许写操作，误动作成本很高。
- 反馈闭环缺失：执行了没有、结果如何，没人知道。

这些问题的本质是：proactive 系统需要把触发、决策、执行、反馈拆开，而不是把模型直接暴露给事件流。

## 做法 / 步骤

### 1. 先做事件归一化

不要让模型直接消费原始 webhook。建议在 OpenClaw 前加一个轻量事件归一化层，把 GitHub issue、Prometheus alert、定时任务、文件变更统一成结构化事件：

```json
{
  "id": "gh_issue_123",
  "type": "issue.opened",
  "source": "repo:org/service",
  "title": "Deploy failed in staging",
  "body_summary": "Error: connection refused on port 5432",
  "timestamp": "2025-01-01T10:00:00Z"
}
```

其中 `id` 作为幂等键，后续去重和审计都靠它。

### 2. 动作封装成 MCP 工具

在 OpenClaw 里，每个外部动作最好封装成一个 MCP tool，而不是让模型自由写 shell。例如：

- `get_issue`
- `comment_issue`
- `query_service_logs`
- `restart_deploy`
- `send_alert`

工具描述里要明确前置条件和副作用。模型只负责从少量工具中选择，不直接接触底层命令。这样即使模型判断失误，破坏面也有限。

### 3. 设置决策与审批策略

触发后启动一个受限 task，内部先读取事件上下文，调用只读工具收集信息，再输出动作建议。策略层可以这样定义：

```yaml
policy:
  read_only: auto
  reversible_write: dry_run_or_confirm
  high_risk: require_approval
  default_when_uncertain: no_action
```

模型输出需要包含 `action`、`reason`、`confidence` 三个字段，代码层根据策略决定是直接执行、生成计划通知用户，还是等待批准。

### 4. 写回与冷却

动作执行后，把结果写回事件源。比如 issue 自动分类后，用 `comment_issue` 写一条简短说明；告警处理完后更新 annotation。同时记录审计日志，并对同一 `event_id` 做去重，冷却窗口建议 5–15 分钟。

## 踩坑点

- **触发风暴**：webhook 重试、状态多次变化都会导致同一事件反复触发。没有幂等键一定会乱。
- **上下文膨胀**：把几百行日志直接丢给模型，不仅费 token，还容易误判。先做摘要和关键行提取。
- **静默失败**：proactive 任务失败后如果没人知道，可能造成“以为它办了”的假象。失败通知要和任务本身解耦。
- **权限过大**：不要给 proactive agent 默认写权限。动作白名单 + 最小权限是底线。
- **假阳性**：模型在不确定时倾向“做点什么”，所以策略里要显式设置 `default_when_uncertain: no_action`。

## 可复用建议

- **第一阶段只做“检测 + 通知”**：让 agent 只读数据、生成摘要和建议，由人确认后再逐步放开写操作。
- **每个 proactive 任务配 runbook**：写清触发条件、允许动作、回滚步骤、负责人。没有 runbook 的自动任务，迟早变成黑盒。
- **审计日志至少包含**：事件 ID、决策理由、工具调用参数、执行结果、耗时。出事时能回溯。
- **dry-run 模式很实用**：调试阶段可以只看计划动作不执行，确认模型行为符合预期后再放开。
- **MCP server 侧做校验和限流**：不要把所有约束都交给模型，工具层要能拒绝非法参数和超频调用。

## 总结

Proactive 不是让模型更“主动”，而是让系统在正确边界内自动响应。工程上比较稳的路径是：先把事件归一化，再用 MCP 工具限制动作面，通过策略层控制执行权限，最后补上审计和冷却。从只读通知做起，比一步到位安全得多。

---

