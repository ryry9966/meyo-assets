---
title: Proactive Agent 实作：用 OpenClaw 定时巡检 + MCP 推送到群聊
feedId: 32553
source: 综合讨论
publishedAt: 2026-08-11
---

## 1. 背景：为什么需要 proactive 能力

大部分 AI 助手仍然是**被动响应式**的：你问一句，它答一句。即使接入了 MCP 或插件生态，多数场景依然依赖人工发起对话。  
在自动化运维、团队协作、个人效率领域，我们真正缺的不是“问一句才动”的能力，而是 **“事件发生 → 自动感知 → 主动通知”** 的闭环。

Proactive 能力，可以让 Agent 没等你开口就把事办了——  
以我最近在 OpenClaw 上的实践为例：每天定时巡检 GitHub 仓库的 Open PR，发现有超过 24 小时未 review 的 PR 就自动推送到飞书群，省去了人工翻页查询。

## 2. 问题拆解

要让 Agent 具备 proactive 行为，需要解决三个工程问题：

1. **调度触发**：什么时机让 Agent 执行任务？cron 还是事件监听？
2. **上下文注入**：怎么让 Agent 知道自己该查什么、推送到哪里？
3. **动作执行**：怎么把结论发到外部平台（钉钉、飞书、Slack）？

在 OpenClaw 体系里，这三个环节分别对应：

- Agent 的 `schedule` 配置（或外部 cron 触发 API）
- 通过 MCP 服务器拉取外部数据（如 GitHub API）
- 通过另一个 MCP 服务器或 Webhook 推送消息

下面是我从零搭建一个 **“GitHub PR 巡检 + 飞书推送” proactive agent** 的全过程。

## 3. 做法与步骤

### 3.1 OpenClaw Agent 核心配置

在 OpenClaw 中创建一个专用 Agent，设定它的 **系统指令** 和 **工具权限**。  
关键点是让这个 Agent **只服务于巡检任务**，避免干扰主对话流程。

```yaml
# agent-config.yaml
name: pr-inspector
schedule: "0 */2 * * *"  # 每两小时执行一次
tools:
  - mcp: github-reader
  - mcp: feishu-sender
memory: persistent
system_prompt: |
  你是一个项目巡检助手。每次被唤醒时，请使用github-reader工具检查仓库owner/repo的待处理PR。
  找出所有created_at超过24小时且尚未reviewed的PR。
  按author分组，生成简洁摘要，通过feishu-sender发送到指定webhook地址。
```

`schedule` 字段是 OpenClaw 在 0.3.x 版本支持的 cron 表达式，它会定时将当前时间作为隐式用户消息唤醒 Agent。

### 3.2 MCP 工具链搭建

**github-reader (MCP server)**  
我用了社区现成的 `@anthropic/mcp-server-github`（需自行生成 GitHub token），暴露 `search_pull_requests` 工具。为了避免每次调用都被限流，配置了缓存参数。

**feishu-sender (MCP server)**  
这个更简单，就是一个 HTTP POST 封装。我用 Python 写了一个轻量 MCP 服务器，暴露 `send_feishu_message` 工具，参数包括 webhook URL 和 Markdown 内容。  
注意，MCP 工具返回必须结构化，否则 Agent 理解困难。

### 3.3 调试与踩坑记录

**坑1：首次唤醒丢失上下文**  
OpenClaw 的 schedule 触发是一次**无状态调用**，Agent 默认不携带历史记忆。如果巡检逻辑里需要上一次运行结果的对比（比如“只有新出现的未 review PR 才推送”），必须启用 `persistent: true` 并且利用 Agent 的 `state` 对象显式读写。否则每次推送会变成全量通知轰炸。

**坑2：时间理解不一致**  
Agent 拿到的“当前时间”是触发时刻的 UTC 时间字符串，但 GitHub API 返回的 `created_at` 也是 UTC。系统提示里要显式要求用 UTC 做比较，否则 Agent 可能按本地时区换算，导致 24 小时判断偏移。

**坑3：MCP 工具调用链过长导致超时**  
github-reader 返回的 PR 列表可能很大。如果 Agent 试图一次性对每个 PR 做详细调用，会在 schedule 窗口内超时。解法是在 system_prompt 里引导 Agent 使用 **批量查询** 模式，并限制返回条数（例如 top 10），必要时让 Agent 输出“剩余 PR 未列出”的提示。

**坑4：飞书 MCP 的 webhook 硬编码**  
为安全起见，webhook URL 不要写在 system_prompt 里，可以作为 Agent 的环境变量注入到 feishu-sender MCP 的配置中。工具函数内部读取环境变量，Agent 只看到工具签名里的“发送到群”语义。

## 4. 可复用建议

这套 proactive 模式可以抽象成 **“定时巡检 Agent 模板”**，我总结的复用公式：

- **数据源**：任意 MCP 服务器（日历、数据库、CI 系统、天气 API）
- **触发条件**：cron / Webhook / 消息队列事件
- **动作**：消息推送 MCP / REST API / 创建工单

你可以把上面的 github 巡检逻辑换成：

- 每天 8 点通过天气 MCP 拿当日天气，若下雨则推送带伞提醒
- 每小时检查数据库慢查询日志，发现增加趋势主动告警
- 监听 CI Webhook 事件，构建失败时直接分析日志并通知对应提交者

核心经验是：**让 Agent 保持在“短生命周期、单一职责”模式**，不要一个 Agent 既当聊天机器人又当巡检员。分离 Agent，反而更容易维护和调试。

## 5. 总结

Proactive 不是玄学，本质是 **调度器 + 工具链 + 合适的任务指令**。  
在 OpenClaw 体系下，结合 MCP 的标准工具接口，可以让 Agent 从“对话机器人”变成真正在后台帮你跑腿的自动化助手。

目前仍存在的局限性：

- 任务编排能力较弱，无法做复杂的条件分支（可以用多 Agent 协作缓解）
- 错误重试与通知风暴抑制需要自己在 MCP 层实现
- 时间敏感型场景需要关注 schedule 的最小粒度（目前 OpenClaw 最小 5 分钟）

如果你也在探索让 AI 帮你主动干活的玩法，欢迎从一个小巡检任务开始，踩一遍坑后你会发现，原来的被动提问模式确实有点原始了。

---

