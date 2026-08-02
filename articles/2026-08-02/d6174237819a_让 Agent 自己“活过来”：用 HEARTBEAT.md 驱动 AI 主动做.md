---
title: 让 Agent 自己“活过来”：用 HEARTBEAT.md 驱动 AI 主动做事
feedId: 31345
source: 综合讨论
publishedAt: 2026-08-02
---

# 让 Agent 自己“活过来”：用 HEARTBEAT.md 驱动 AI 主动做事

## 问题：Agent 像个“算盘珠子”，拨一下才动一下

在 OpenClaw、MCP 插件或自定义 Agent 的日常使用中，一个典型的模式是：用户发起指令 → Agent 执行 → 返回结果。这种模型在处理单次查询时工作良好，但一旦需要 Agent 持续关注某个目标、跨多个上下文窗口执行长线任务，或者在没有人类干预时自行推进项目，它就显得非常被动。

举个例子：你让 Agent 监控一个 GitHub 仓库的 issue 列表，每天早上 9 点自动汇总新增的问题。如果仅靠用户自己每次提醒，这个 Agent 就变成了需要人伺候的工具，而非助手。本质问题在于——大多数 Agent 缺少一个“心跳机制”，没有一个持久的、可自我迭代的待办与状态文件，让它知道自己该主动做什么。

## 背景：HEARTBEAT.md 的灵感来源

在自治 Agent 的设计中，研究者和实践者曾尝试类似 `TODO.md` 或 `MEMORY.md` 的方案，让 Agent 在对话间保持上下文。但这些文件通常只是被动记录，而不是主动触发行为。

**HEARTBEAT.md** 则是一个持续更新的、包含目标、计划、当前状态和待执行动作的活动文件。它的核心理念是：当 Agent 每次被唤醒（无论是通过定时 cron、消息回调，还是用户主动打开对话）时，首先读取自己的 HEARTBEAT.md，然后根据文件中的“下一步动作”自主执行任务，更新状态，最后再写回文件。这样就形成了一个闭环：Agent 自己维护任务队列，自己决定何时行动。

OpenClaw 的插件环境允许 Agent 访问文件系统、调用外部 API，这为 HEARTBEAT.md 的落地提供了基础。我们可以在本地或 Workspace 根目录放置一个规范的 Markdown 文件，由 Agent 通过 MCP 工具或 File System 插件自行读写。

## 做法：如何为 Agent 设置 HEARTBEAT.md

### 1. 定义文件结构

一个实用的 HEARTBEAT.md 至少包含以下几个部分：

```markdown
# Agent Heartbeat

## Meta
- 最后心跳时间: 2025-03-22T08:30:00Z
- 心跳间隔: 3600 秒（由外部 cron 决定）
- 状态: active

## Active Goals
1. [每日] 监控仓库 `user/repo` 的 issue 变化
2. [进行中] 整理本周阅读清单并推送到 Notion

## Today's Intents
- [ ] 检查仓库 issue 并生成摘要
- [ ] 抓取 RSS 源最新文章
- [ ] 向 #general 频道发送早安总结

## Log
- 2025-03-22T08:30:01Z: 开始执行 Daily Intents
- 2025-03-22T08:30:20Z: 成功获取 issue 列表，无新增
```

这种结构保证了 Agent 即使重启对话，也能快速恢复状态。

### 2. 让 Agent 理解并遵守 HEARTBEAT

在 System Prompt 中加入明确的指令：

> “你是一个具有自主心跳的 Agent。每次对话开始时，请先读取项目根目录下的 `HEARTBEAT.md`，检查 `Today's Intents`，并逐一完成其中未完成的任务。完成后更新 Log，并将完成任务打勾。如果时间已过当天，重新生成新一天的 Intents，并根据 Active Goals 调整。你的目标是尽可能让自己在没有用户干预的情况下保持推进。”

同时，在 OpenClaw 中可以为 Agent 配备一个工作技能：**heartbeat**。这可以是一个简单的 MCP 工具，封装读取、更新 HEARTBEAT 的操作，并供 Agent 调用。

### 3. 外部触发器：让 Agent 被定时唤醒

仅靠 HEARTBEAT.md 本身还不够，Agent 需要一个“起点”。常用方式：

- **cron job + OpenClaw API**：编写一个每分钟或每小时运行的脚本，向 Agent 所在 Channel 发送一条空消息或“心跳”指令，例如 `!heartbeat`。
- **OpenClaw 插件定时回调**：如果插件系统支持定时任务，直接在其中触发 Agent 读取 HEARTBEAT 并执行。
- **Webhook 集成**：通过 GitHub Actions 或 Zapier，在特定事件（如新 Issue）发生时，推送一条消息到 OpenClaw，Agent 被唤醒并处理。

这里一个关键点：**心跳触发器本身不需要包含具体任务**，它只是给 Agent 一个“该干活了”的信号，具体做什么完全由 HEARTBEAT.md 决定。

### 4. 状态回写与错误处理

Agent 在执行任务时可能会遇到错误、超时或部分失败。HEARTBEAT.md 的 Log 区域需要记录这些细节，并设置重试策略。例如：

- 如果网络请求失败，Agent 在 Intents 旁添加注释：`⚠️ 网络错误，下次重试`
- 连续失败 3 次后，将任务标记为 `[blocked]` 并通知用户

同时，主动 Agent 必须有明确的“安全停止”机制。我们可以在 Meta 区域设置一个 `max_actions_per_heartbeat` 限制，防止 Agent 进入死循环。

## 踩坑点

1. **Agent 幻觉导致 HEARTBEAT 被污染**  
   某些情况下 Agent 可能会写入不存在的目标或虚构的 Log。解决办法是：为 HEARTBEAT.md 添加 JSON Schema 校验（通过 MCP 工具实现），每次写入前检查格式。

2. **心跳过于频繁，导致 Token 快速消耗**  
   如果每 5 分钟就触发一次心跳，Agent 会重复检查大量已完成任务。建议以小时为单位设定心跳，并让 Agent 先判断当前时间与上次执行的差异是否满足最低间隔。

3. **多会话竞争写文件**  
   如果在不同 Channel 同时触发心跳，可能同时写入 HEARTBEAT.md，出现冲突。可以利用文件锁或 OpenClaw 的单实例工具访问机制避免。简单方案是：只在主会话中运行心跳 Agent，其他会话通过 IPC 通知它。

4. **任务积压**  
   如果 Agent 由于 API 限制无法一次完成所有 Intents，需要在 HEARTBEAT 中记录进度（如处理到第几页），下次心跳从中断处继续。

## 可复用建议

- **将 HEARTBEAT.md 看作“Agent 的工作台”**，而不仅仅是待办列表。它能记录上下文缓存、临时变量、甚至思维链片段。
- **结合 MCP 的记忆模块**：可以把 HEARTBEAT 中的关键事实同步到记忆插件中，让 Agent 在跨心跳时保持长期上下文。
- **逐步赋予自主权**：初期可以让 Agent 只执行“读取并报告”类任务，验证稳定性后，再开放“修改仓库内容”、“发送消息”等写操作。
- **保留人工入口**：即使用户不直接干预，也可以随时向 Agent 发送指令，让它更新 Active Goals。HEARTBEAT 不排斥人类协同。

## 总结

HEARTBEAT.md 并不是一个复杂的技术，但它改变了 Agent 的运作范式——从“应答式”转向“主动式”。对于长期运行的个人助手、监控机器人或自动化工作流，这种自主心跳机制能极大地提升效率和可靠感。

在 OpenClaw 体系下，通过 System Prompt 指导 + MCP 工具封装 + 外部触发器，我们可以让 Agent 真正“活过来”，每天按时完成预设任务，甚至在没有人类监督时自行探索和优化工作流程。未来，结合更智能的规划能力，HEARTBEAT.md 也许能演进为 Agent 的“自主神经系统”。

---

