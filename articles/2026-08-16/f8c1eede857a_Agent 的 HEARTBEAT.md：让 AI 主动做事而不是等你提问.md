---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33371
source: 综合讨论
publishedAt: 2026-08-16
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

## 背景：Agent 为什么总是“等你开口”

很多 OpenClaw 用户在配置好 Agent、接好 MCP、写完插件后，会进入一个尴尬期：工具很全，能力很强，但 Agent 仍然像个高级问答机器人。你问一句，它动一下；你不问，它就一直沉默。

问题不在于模型不够聪明，也不在于 MCP 工具不够多，而在于 Agent 缺少一个“主动触发机制”。人类助理之所以能主动做事，是因为他们知道什么时候该看一眼待办、什么时候该跟进项目、什么时候该提醒风险。而我们的 Agent 默认没有这个“节奏感”。

一些用户会尝试用 cron 定时任务，但很快发现单纯定时触发没有上下文：Agent 被唤醒后不知道要做什么，只能发一句“我在，有什么需要？”这显然不是真正的主动。

## 问题：被动 Agent 的三个实际代价

1. **上下文丢失**：你上周让 Agent 关注某个 issue，三天后它已经“忘”了要主动回看。
2. **任务延迟**：依赖你手动触发，等于把“记得让 AI 做事”变成了一项新的待办。
3. **资源浪费**：配置了大量 MCP 工具，但大部分时间闲置，因为没有人告诉 Agent 何时使用它们。

我尝试过不少方案：在 system prompt 里写“请主动检查”、用 cron 定时发消息、用外部任务队列。最后发现最稳定、最可维护的做法，是给 Agent 一个显式的“心跳文件”。

## 做法：用 HEARTBEAT.md 定义主动节奏

HEARTBEAT.md 是一个放在 OpenClaw 工作区根目录的 Markdown 文件。它的作用不是存储知识，而是定义 Agent 的主动检查节奏和动作清单。

### 第一步：创建 HEARTBEAT.md

一个最小可用的结构如下：

```markdown
# HEARTBEAT

## Cadence
- 每个工作日 09:00 检查一次
- 每个工作日 17:00 检查一次
- 周末只检查一次（如果工作区有未完成事项）

## Check
1. 读取 `AGENDA.md`，列出今天计划内事项
2. 读取 `FOLLOWUPS.md`，查看是否有超过 24h 未跟进的标记
3. 读取 `BLOCKERS.md`，确认是否有阻塞项需要外部信息

## Actions
- 如果 FOLLOWUPS 中有 overdue 项，生成一条简明提醒，发送到通知渠道
- 如果 AGENDA 中有需要外部查询的事项（如接口状态、依赖版本），主动执行查询
- 如果 BLOCKERS 中有超过 48h 未更新的项，询问负责人（或通过 IM 通知）

## Report
将检查结果追加到 `HEARTBEAT_LOG.md`，格式：
- 时间戳
- 检查到的问题
- 是否已采取行动
- 需要用户决策的事项

## Limits
- 只读访问上述文件，除非 Action 中明确允许写
- 不修改 BLOCKERS.md 中的原始描述
- 不发送两次相同内容到通知渠道
```

### 第二步：让 Agent 在每次会话启动时读取

在 OpenClaw 的 agent 配置中，把 HEARTBEAT.md 作为启动时必读文件。这样即使不是定时触发，你打开会话时 Agent 也会先检查是否有需要主动报告的事项。

具体做法可以是：

- 在 system prompt 中加一句：“会话开始时，先读取工作区根目录的 HEARTBEAT.md，并按其 Cadence 判断当前是否处于检查窗口。”
- 或者用 OpenClaw 的 memory/skill 机制，将 HEARTBEAT.md 注册为持久上下文。

### 第三步：用 cron 触发“心跳检查”

光靠会话启动还不够。真正实现“主动做事”，需要配合 cron。以 OpenClaw 的定时任务能力为例：

```yaml
cron:
  - name: heartbeat-morning
    schedule: "0 9 * * 1-5"
    action: "read HEARTBEAT.md, run check and report"
  - name: heartbeat-evening
    schedule: "0 17 * * 1-5"
    action: "read HEARTBEAT.md, run check and report"
```

定时任务触发后，Agent 读取 HEARTBEAT.md 里的 Check 清单，执行 Actions，并写入 HEARTBEAT_LOG.md。

关键点在于：cron 只负责“唤醒”，HEARTBEAT.md 负责告诉 Agent 醒来后做什么。两者职责分离后，就不会出现“定时唤醒了但不知道干嘛”的问题。

### 第四步：逐步扩大主动范围

一开始不要把 HEARTBEAT.md 写得太复杂。可以从一两个检查项开始，比如：

- 检查某个 RSS 源是否有更新
- 检查某个 API 是否返回异常状态码
- 检查本地文件目录是否有新增文件

等 Agent 能稳定执行这些基础心跳后，再逐步加入更复杂的动作，比如自动生成日报、自动创建 issue、自动发送提醒。

## 踩坑点

### 1. 心跳太频繁导致打扰

刚开始很容易把检查频率设置得过高，比如每小时一次。如果 Agent 每次检查都发消息，你会迅速感到疲劳。建议按“是否需要人工决策”来区分：只有需要用户决策时才主动通知，其他情况只写日志。

### 2. 文件读写冲突

如果多个 Agent 实例或定时任务同时读写 HEARTBEAT_LOG.md，会出现内容覆盖或重复记录。可以在写入前检查文件末尾，或使用追加模式。更稳的做法是用一个独立的 `heartbeat.lock` 文件做简单互斥。

### 3. 上下文膨胀

Agent 每次会话启动都读取 HEARTBEAT.md、AGENDA.md、FOLLOWUPS.md、BLOCKERS.md，如果这些文件越来越大，上下文窗口会被挤占。建议定期归档历史记录，保持这些文件在几十行以内。

### 4. 状态污染

Agent 有时会“好心”修改 AGENDA.md 或 FOLLOWUPS.md 的格式，导致下一次读取困难。一定要在 HEARTBEAT.md 的 Limits 里明确哪些文件只读、哪些可以追加。

## 可复用建议

- **把 HEARTBEAT.md 当作接口而不是提示词**：它的内容应该结构化、可解析，而不是一段散文。Agent 需要快速定位到 Cadence、Check、Actions、Limits。
- **和日历/任务工具集成**：如果 OpenClaw 已经接入了日历 MCP 或任务管理 MCP，让心跳检查结果和这些工具联动，而不是只写 Markdown 日志。
- **给心跳一个“静默模式”**：在 HEARTBEAT.md 中增加一个 `silent: true` 标记，当你不希望被打扰时，Agent 只记录不通知。
- **版本化你的 HEARTBEAT.md**：这个文件本身也值得纳入版本控制。你会发现它会随着项目节奏不断调整。

## 总结

HEARTBEAT.md 不是什么高级架构，它只是一个简单的工程约定：把 Agent 的主动节奏显式化、结构化、可版本化。它的核心价值在于解决了“Agent 有能力但没时机”的问题。

对于 OpenClaw 用户来说，这个模式几乎零成本：你只需要一个 Markdown 文件、一个 cron 定时任务，以及几行 system prompt 修改。但一旦跑通，你的 Agent 会从“等指令的工具”变成“会看时钟的协作者”。

真正有用的自动化，不是你能随时召唤 Agent，而是 Agent 在合适的时间带着有价值的信息主动出现。

---

