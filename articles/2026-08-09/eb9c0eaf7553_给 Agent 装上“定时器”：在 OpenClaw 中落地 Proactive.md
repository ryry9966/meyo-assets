---
title: 给 Agent 装上“定时器”：在 OpenClaw 中落地 Proactive 能力的工程记录
feedId: 32227
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：Agent 的默认象限是“应答式”

大部分基于 LLM 的 Agent（包括我们常用的 OpenClaw 智能体），工作模式是等待用户输入，然后调用工具完成推理与执行。这种 reactive 的范式适合对话场景，但在运维巡检、信息监控、周期报表等场景下，期望 Agent 能**自行感知时间推进或外部事件变化，并在不依赖人工触发的前提下完成一组动作**。这就是“proactive 能力”——Agent 成为数字化员工，而不只是高级聊天工具。

我们团队一直在用 OpenClaw 做自动化实践，最近尝试把部分日常巡检（如监控 GitHub 仓库中特定标签的未分配 Issue）交给 Agent 自主处理。下面复盘一下这个过程、踩过的坑和可复用的模板。

## 问题定义：从“人找事”变成“事找人”

具体需求：

- 每 30 分钟检查一次 GitHub 仓库 `backend` 中的 `bug` 标签 Issue。
- 如果 Issue 没有 assignee，且标题包含「P0」或「crash」，需要自动分配给值班同学，并在 Slack 特定频道发通知。
- 处理过的 Issue 不应重复分配或重复通知。

核心挑战：

1. **触发机制**：Agent 不会被用户唤醒，需要定时启动。
2. **状态记忆**：必须记住哪些 Issue 已经处理，避免重复动作。
3. **决策稳健**：LLM 容易“自作主张”，需要足够的约束，防止误分配或信息幻觉。

## 做法：用 OpenClaw + MCP 构建一个 Proactive Agent

### 1. 工具准备：封装 GitHub 操作为 MCP 服务

OpenClaw 社区对 MCP 的支持比较成熟，我们起了一个 Python MCP 服务，暴露三个工具：

- `search_issues(label, state)`：返回未关闭 Issue 列表（含标题、ID、assignee 等）。
- `assign_issue(issue_id, username)`：将 Issue 分配给指定用户。
- `comment_issue(issue_id, body)`：添加评论，用于记录自动分配的原因。

另外通过 Slack MCP（社区已有现成的）暴露 `send_slack_message(channel, text)` 工具。

**关键点**：工具返回结构一定要简单、确定性强，方便后续 Prompt 约束。例如 `search_issues` 只返回必要字段（id、title、assignee 是否为空），不要返回整个 Markdown 正文，避免 Token 膨胀和 LLM 迷失。

### 2. Agent 设计：行为约束全写在 Prompt 里

在 OpenClaw 中新建 Agent，System Prompt 大致如下：

```
你是一个仓库巡检智能体，仅在接收到“执行巡检”命令时运行。现在你会被调用。
你可以使用以下工具：
- search_issues
- assign_issue
- comment_issue
- send_slack_message

巡检规则：
1. 使用 search_issues 获取所有标签为“bug”且状态为“open”的 Issue。
2. 逐一检查每个 Issue：
   - 如果 assignee 不为空，跳过。
   - 如果 title 包含“P0”或“crash”，则必须分配给值班人“oncall-dev”，并添加评论“系统自动分配 P0/crash 问题”。
   - 分配成功后，使用 send_slack_message 发送通知到 #ops-alert，包含 Issue 标题和链接。
3. 已经处理过的 Issue 不需要再操作，我会通过外部机制告诉你不要重复。但你每次仍要检查 assignee 为空的情况，因为 Issue 可能在你的外部记忆没有被标记但已被人工分配。
```

这个 Prompt 明确给出了决策分支和“必须”、“跳过”等边界词，大幅降低幻觉空间。我们没有把“已处理 ID”放在 Prompt 里，因为历史 ID 会越来越长，容易撑爆上下文。我们把状态外置。

### 3. 状态去重：引入轻量的外部 KV 存储

用一个 Redis Key `processed_issue_ids` 存储已经处理过的 Issue ID（Set 结构）。在 OpenClaw **运行前**，外层的调度脚本查询 Redis，把“本次巡检应当忽略的 Issue ID”以额外环境变量注入 Agent 的工具描述中，或直接在调用 `assign_issue` 时由 MCP 工具内部检查 Redis 决定是否实际执行。

我们的实现选择后者：在 MCP 的 `assign_issue` 工具里加入去重逻辑：

```python
if issue_id in redis_client.smembers('processed_issue_ids'):
    return {"status": "skipped", "reason": "already processed"}
# 执行分配后
redis_client.sadd('processed_issue_ids', issue_id)
```

这样做的好处是**工具层强一致**，即使 LLM 决策出错试图分配已处理 Issue，实际操作也被拦截。这是防御性设计。

### 4. 定时触发：外部 cron + OpenClaw API

OpenClaw 提供了通过 API 触发 Agent 运行的能力（部分部署环境也内置 Scheduled Triggers）。我们使用一个简单的 cronjob，每 30 分钟向 OpenClaw 的这个 Agent 会话发送一个固定指令：“执行巡检”。Agent 收到后按规则运行一次。

需要注意：每次触发应使用**独立的新会话**，避免历史消息积累引发上下文错乱。我们通过 `session_id` 为时间戳生成唯一值来保证。

### 5. 通知与日志

分配成功后，Slack 消息包含 Issue 标题、链接、分配人和时间戳。MCP 工具返回详细的操作日志，我们把它推送到 Loki，便于回溯 Agent 在几点几分做了什么决策，方便排错。

## 踩坑点实录

**坑 1：定时频率与 GitHub API 限流**
在测试时，我们把 cron 设得过密，加上 search_issues 每次拉取全量 open Issue，很快就触发了 GitHub 的二级限流。后来加了调用缓存：30 分钟内如果 cron 多次执行（比如手动测试），MCP 工具对同一查询参数直接返回缓存结果。生产上保持 30 分钟间隔完全足够。

**坑 2：LLM 的“过度殷勤”**
早期的 Prompt 没有明确说“分配成功后不需要重复通知”，导致 Agent 在第二轮巡检时又对已分配的 Issue 发了一次 Slack 消息（尽管 assignee 已不为空被跳过，但它依然执行了通知）。修复方式：在 Prompt 中明确规定“只有本次分配成功的 Issue 才需要发 Slack 通知”。

**坑 3：MCP 返回值漂移**
某次修改 MCP 工具的返回结构，增加了 `status` 字段，但没更新 Agent 的 Prompt，导致 LLM 对 `"status":"already_assigned"` 的理解出现偏差，认为需要再次尝试分配。教训：工具描述保持稳定或采用 JSON Schema 强约束。

**坑 4：时间窗口导致的“漏网之鱼”**
如果 Issue 是在两次巡检之间被人工分配掉，检索时 assignee 已经不为空，Agent 会正确跳过，没问题。但我们最初逻辑是“记录所有检索到的 Issue ID 为已处理”，这会导致后面真正的未分配 Issue（与之前某个已关闭 Issue 同 ID？不可能，ID 唯一）不会被漏掉。我们没有犯这个错，但设计去重逻辑时确实要区分“处理过”和“见到过”。

## 可复用建议：Proactive Agent 模板

经过几次迭代，我们总结出一个模版，适用于大部分定时巡检类 proactive 需求：

1. **最小工具集**：
   - 查询当前状态（可能有多条记录）
   - 执行某项动作（分配、关闭、发通知）
   - 内部包含幂等/去重检查的外部状态存储（Redis/DB）
2. **Prompt 工程套路**：
   - 明确列出判断条件和分支动作
   - 使用“必须 / 禁止 / 跳过”等术语
   - 明确每条记录的预期输出（例如“成功分配 N 个 Issue”）
3. **触发与隔离**：
   - 使用 cron 或计划任务触发，每次独立会话
   - 通过 `X-Idempotency-Key` 之类的机制确保不会并发执行两次（可选）
4. **可观测性**：
   - MCP 工具记录每次实际状态变更
   - Agent 输出结构化日志到集中日志系统
   - 设置异常通知（例如调用失败直接报 Slack）

## 总结

Proactive 能力并不是让 Agent 一直“醒着”，而是通过**定时触发 + 边界清晰的决策 + 外部状态防重**，让 AI 可以在无人值守的情况下完成确定性的、有限的自动化任务。这套方案已经在我们的巡检场景平稳运行了两周，每天减少人工处理约 6-8 个重复工单分配操作。如果不是完全依赖于 LLM 做关键判罚，而是把 LLM 作为流程中的条件过滤和内容生成组件，proactive 模式会非常可靠。

对于 OpenClaw 社区的朋友，推荐从“最小闭环”开始：连接一个数据源（GitHub、Jira、数据库查询），用 MCP 包装，再写一段严苛的 Prompt，然后挂上 cron，就能真切感受到 Agent 替你跑腿的工程价值。

---

