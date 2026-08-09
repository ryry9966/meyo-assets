---
title: 从被动到主动：用 OpenClaw 构建 Proactive Agent 的工程实践
feedId: 32279
source: 综合讨论
publishedAt: 2026-08-09
---

# 从被动到主动：用 OpenClaw 构建 Proactive Agent 的工程实践

## 背景
绝大多数 AI 助手都是“提问-回答”的被动模式。但在自动化运维、业务流程监控、智能提醒等场景中，我们需要 Agent 能在条件满足时**主动介入**，不等用户开口就把事情办了。这种 **proactive 能力** 正在成为 Agent 落地的重要拼图。

OpenClaw 生态提供了足够灵活的插件与 MCP 机制，让工程师可以自己“拼”出一套可控的主动触发体系。本文记录一次将 cron 触发、MCP 工具调用和 Agent 上下文化决策结合起来的实战过程，面向已有 OpenClaw、MCP 及插件开发经验的读者。

## 问题拆解
“主动办事”需要解决三个核心问题：
1. **触发源**：什么事件或时间规律能启动流程？
2. **决策**：Agent 如何根据当前上下文判断该不该动、动什么？
3. **安全执行**：如何限制主动行为的作用域，防止幻觉产生破坏性操作？

我们以一个实际场景为例：**每早 9 点自动分析待处理的 GitHub Issues，对超过 3 天未活动的 issue 自动评论提醒负责人，并生成摘要发送到 Slack**。

## 做法与步骤
### 1. 设计触发链路
放弃在 Agent 内部构建复杂定时器，采用“外部触发 + 内部执行”的松耦合模式。

- 使用 Linux cron 或 Kubernetes CronJob 定时发送一个 HTTP POST 到 OpenClaw 的 `/api/trigger` 端点（需自行在 OpenClaw 插件层实现一个轻量 HTTP handler）。
- 该端点仅做一件事：向指定的会话（session）写入一条预定义的系统消息，例如 `trigger:morning-issue-check`。
- 这样不破坏对话历史，只作为一个外部信号注入。

如果你的 OpenClaw 实例托管在本地，也可以用 `curl` + 内网地址触发，足够可靠。

### 2. 注册 MCP 工具，赋予执行能力
Agent 需要工具来查询问题列表、发送评论和推送 Slack。通过 MCP server 暴露三个工具：

- `list_stale_issues(repo, days)`: 调用 GitHub API 返回超过指定天无更新的 issues。
- `add_issue_comment(issue_id, body)`: 在指定 issue 下添加评论。
- `send_slack_summary(text)`: 将 Markdown 摘要推送到 Slack 频道。

MCP server 的实现用 Python `FastMCP` 几分钟就能完成，注意处理好 GitHub token 和 Slack webhook 的环境变量注入。测试时可以直接用 MCP 的调试工具调用验证，确保每个工具独立可用。

### 3. 构建 Agent 决策逻辑
在 OpenClaw 中创建一个专门用于“早晨巡检”的 Agent 配置，设定 system prompt 如下关键点：

- 当看到 `trigger:morning-issue-check` 时，立即调用 `list_stale_issues`，提取 3 天以上未活动的 issues。
- 对于每个 stale issue，生成礼貌但明确的提醒评论，调用 `add_issue_comment` 执行。
- 完成后汇总本次处理数量、结果，调用 `send_slack_summary` 汇报。
- **重要**：任何工具调用失败时必须输出错误报告，而不是自行臆造执行结果。

通过“注入触发信号 -> Agent 解析 -> 工具调用”的流水线，Agent 只在收到明确标记时才展开行动，避免日常对话中误触发。

### 4. 主动行为的边界控制
所有有写效果的工具（如发表评论、发消息）都要在 MCP 层做一层额外校验：

- `add_issue_comment` 禁止在已关闭的 issue 上操作。
- `send_slack_summary` 限制每天最多调用 3 次，避免代码错误导致消息风暴。
- 工具返回前记录日志，包含调用者 session ID、时间戳，方便审计。

这些校验写在 MCP server 内部，不依赖 Agent 本身的判断，形成“双重保险”。

## 踩坑点
- **重复触发**：cron 任务可能因为网络超时重试，导致同一个触发消息被注入多次。解决方案是在消息体中携带唯一 `trigger_id`，Agent 端用 Redis 记录最近 5 分钟内的已处理 ID，去重处理。
- **Agent 幻觉**：在没有获取到 issues 列表时，Agent 可能编造处理结果。解决办法是强制要求“必须先成功调用 `list_stale_issues`，再执行后续动作”，并在 system prompt 中加入明确规则，同时监控日志中工具调用链的完整性。
- **资源浪费**：如果 cron 频率过高，OpenClaw 会话中会堆积大量无意义的系统消息。建议 cron 最小间隔 ≥5 分钟，并在触发端点做频率限制（如每会话每分钟最多 1 次）。
- **时区与工作日**：cron 表达式需要匹配业务时区，并考虑节假日抑制。可以在触发端点代码中增加简单的日期判断，工作日才真正注入触发消息。

## 可复用建议
- **模式提炼**：将“外部触发 → 会话信号 → Agent 决策 → MCP 工具执行”抽象为一个 Proactive Runtime 模块，支持通过 YAML 定义触发条件、目标 Agent 和参数。
- **渐进式开启**：使用 feature flag 控制主动特性，先在少量会话中灰度，观察行为是否符合预期。
- **人机协同**：对高风险操作（如删除资源、合并代码）可以设计为“Agent 生成建议 + 等待用户确认”的半主动模式，而不是完全自动执行。
- **监控与告警**：对所有主动发起的工具调用次数设置阈值监控，异常飙升立即通知运维。

## 总结
Proactive 能力让 Agent 从一个被动的“问答机”进化为真正的“数字巡检员”。通过 OpenClaw 的插件扩展 + MCP 工具 + 外部调度器的组合，我们可以用较低的成本构建出“不等你开口就把事办好”的自动化流程。关键点在于：触发通道清晰、决策上下文严格、执行边界牢固、可观测性完善。如果你已经在用 OpenClaw 做日常自动化，不妨从一个小范围巡检任务开始，让 Agent 主动一次。

---

