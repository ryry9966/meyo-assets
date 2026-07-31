---
title: 让 AI 助手不等开口就把事办了：基于 MCP 的 Proactive Agent 工程实践
feedId: 31130
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

绝大多数 AI 助手的工作模式是请求-响应：你给它一个指令，它执行并返回结果。这种被动模式在处理突发、周期性或条件驱动的任务时非常吃力，比如“每天早上 8 点整理未读邮件摘要”“当部署流水线失败时自动分析日志并创建 issue”。要实现这类“不等你开口就把事办了”的能力，就需要让 agent 具备主动（proactive）执行的能力。

在 OpenClaw 生态里，借助 MCP（Model Context Protocol）的插件化能力和系统级调度，我们可以相对安全、可观测地把 proactive agent 落地。本文将给出一个完整的工程实践——通过 MCP 定时触发器 + 通知插件，让 agent 在固定时间点主动抓取信息、生成摘要并推送到通讯工具。

## 问题拆解

要让 agent 主动干活，需要解决三个关键点：

1. **可靠的触发机制**：用什么来启动 agent，而不是人等机。
2. **可观测的任务执行**：自动运行不是“放生”，需要知道每次跑没跑、结果如何。
3. **失败的收敛与约束**：自动任务一旦出错，不能静默失败，也不能刷屏式报警。

如果不加控制，proactive agent 很容易变成故障源头：死循环消耗 token、错误重复通知、权限过大导致意外操作。

## 实现步骤

以下方案基于 OpenClaw + 一个提供 cron 能力的 MCP server（示例中使用 `cron-trigger` 这类社区插件）。整体流程：

**环境准备**
- OpenClaw（0.6+ 版本，支持 MCP）
- 一个 MCP cron 服务，负责根据 crontab 表达式触发 agent
- 一个通知 MCP server（如企业微信机器人、飞书、Slack）
- 需要主动执行的任务：例如调用 GitHub API 获取最近流水线状态，再让 LLM 总结

**1. 注册 MCP 服务**

在 OpenClaw 的 `config.yaml` 中注册两个 MCP 后端：

```yaml
mcp_servers:
  - name: cron-trigger
    command: npx -y @anthropic/mcp-server-cron  # 仅为示例
    args: []
  - name: wecom-bot
    command: /path/to/wecom-mcp-server
    env:
      WECOM_WEBHOOK_URL: ${WECOM_WEBHOOK_URL}
```

**2. 编写任务 agent**

创建一个 `proactive_github_summary` agent，skill 定义为：
- 通过 GitHub API 拉取指定仓库最近 10 条 Actions 记录
- 筛选出失败或取消的流水线
- 让 LLM 生成简要故障分析和修复建议
- 通过 wecom-bot 推送到群

在 agent 的 prompt 里强调：不要做任何破坏性操作（只读），输出格式为 markdown。

**3. 设置定时触发**

在 cron-trigger 的配置中（可以是单独配置文件或者通过 MCP 工具调用）添加计划：

```json
{
  "name": "morning-pipeline-check",
  "schedule": "0 8 * * 1-5",
  "target_agent": "proactive_github_summary",
  "input": { "repository": "org/repo" },
  "options": {
    "timeout_seconds": 180,
    "max_retries": 2,
    "retry_delay_seconds": 60
  }
}
```

这里 `target_agent` 是 OpenClaw 中注册的 agent ID，cron-trigger 会在匹配时间点向 OpenClaw 发起一次内部调用，相当于模拟用户发消息。

**4. 日志与监控**

建议在 agent 内显式输出一条结构化日志，OpenClaw 会将每次运行为一个 session，可以通过 session 列表回溯。在最终通知中，除了摘要内容，还要附上本次运行的 `trace id`，方便事后排查。

## 踩坑点

- **时区问题**：cron 表达式默认可能是 UTC，需要确认 MCP cron server 所在容器的时区，或显式使用带时区的 cron 库。否则你以为的早上 8 点可能是下午 4 点。
- **任务重叠**：如果上一次运行超过 1 小时还没结束，下一次触发时怎么办？cron-trigger 需要支持 skip 或 delay 策略，切记不能无脑并发，否则可能打爆 API 限流或产生重复消息。
- **敏感信息**：Agent 自动运行必须使用凭据（GitHub token、webhook URL），不能硬编码在配置中。全部通过环境变量注入，并配合 OpenClaw 的 secrets 管理。
- **静默失败**：自动任务失败后，如果没有感知，等于没做。我们让 agent 在异常时也发送一条简单告警，标明失败原因。但要控制发送频率，同一类错误不要连续通知超过 3 次。可以通过一个小型状态文件实现（例如 SQLite 记录上次通知时间）。
- **权限最小化**：自动任务只给予必要的 API 权限。让 agent 能创建 issue，就只用经典 token 并且 scope 限定特定仓库，不要用全局 token。

## 可复用建议

1. **从“建议模式”开始**：在真正自动执行前，先让 agent 产出草稿消息，发送到只有你自己的调试群，人工看三天没问题再切到正式群。
2. **为每个 proactive 任务加唯一 idempotency key**：可以根据任务名 + 触发时间窗口（如精确到小时）生成，避免因重试等原因导致重复动作。
3. **设置运行窗口与熔断**：为任务配置最大运行时长（如 5 分钟），超时强制终止；连续失败 N 次后自动暂停计划，需要人工确认再恢复。
4. **审计所有自动动作**：每次触发的输入、输出、决策过程以 session 持久化，保留至少 30 天。这在出现莫名其妙操作时可以救命。

## 总结

Proactive 能力是 AI 助手从“工具”迈向“协作者”的关键一步。在 OpenClaw+MCP 的组合下，通过可靠的定时触发、可控的任务策略和最小权限原则，我们可以做出安静、守规矩、不等开口就把事情办妥的 agent。但这绝不是简单的“设个 cron 就好了”，需要认真对待失败模式、权限边界和人机协同的节奏。先小范围跑稳，再逐步交给它更多信任。

---

