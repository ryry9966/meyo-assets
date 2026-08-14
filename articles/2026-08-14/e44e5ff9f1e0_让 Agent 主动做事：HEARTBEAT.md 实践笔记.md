---
title: 让 Agent 主动做事：HEARTBEAT.md 实践笔记
feedId: 33123
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

在 OpenClaw/Agent/MCP 的自动化实践里，很多人的配置其实不差：工具接好了，MCP server 能跑通，插件也能触发。但 Agent 仍然像问答机——你问一句，它动一下。更麻烦的是任务一旦跨会话，进度就丢；即使挂了记忆插件，也常常只是被动检索，不会主动推进。

一个典型场景：你让 Agent 每天定时检查某个服务状态并处理异常。结果第二天发现它只在 9:00 被唤醒时看了一眼，什么都没做，也没有留下“为什么没做”的记录。你只能再手动问一遍。

## 问题

核心不是 Agent 不够聪明，而是缺少一个稳定、可轮询、可审计的“触发面”。具体表现为：

1. 没人提问，Agent 就不运行。
2. 状态不可观测：不知道上次做到哪、卡在哪。
3. 主动行为不可控：就算加了定时唤醒，也容易变成“随便找点事做”或空转。

## 做法

把 `HEARTBEAT.md` 放到工作区根目录，作为 Agent 每次被唤醒时必读、必更新的“脉搏文件”。它不替代任务队列或数据库，只解决“下一次醒来该干什么”和“上次为什么停下”。

### 1. 定义最小字段集

不建议一上来就建十几个字段。先跑一周，四个字段足够：

```markdown
# HEARTBEAT.md

## Meta
- owner: agent-7
- last_check: 2025-01-01T09:00:00Z
- interval: 30m

## Top Intent
<!-- 一句话说明当前最重要的目标 -->

## Tasks
<!-- 最多 5 条，状态：todo/doing/blocked/done -->
- [ ] ...

## Blockers
<!-- 阻塞项，带 owner 和期望解决时间 -->
```

稳定后再加 `Decisions` 和 `Signals`。`Decisions` 记录最近 3 条关键决策，避免下次重复争论；`Signals` 放需要人工介入或跨 Agent 协作的信号。

### 2. 设置触发

在 OpenClaw 里可以挂到插件的 cron trigger 上；如果没有现成 trigger，就用系统 crontab 调起 headless 会话，执行固定 prompt。频率建议 30 分钟起步，不要设成每 1 分钟一次。

触发 prompt 示例：

```text
Read HEARTBEAT.md. If there is a `doing` task older than 2*interval, mark it blocked and add a signal. Then pick the first `todo` that is safe and small, execute it using available tools, update status, and append a one-line decision if needed. Do not add more than one new task per run. If nothing is safe, only update last_check and exit.
```

### 3. 限制执行范围

Prompt 里必须写清：只动 `tasks/blockers/signals` 相关；每次最多完成 1 个 todo；新增任务必须写清楚验证方式；不得清空 `Decisions`。这能防止 Agent 把文件改得面目全非。

### 4. 用 git 做变更追踪

把 `HEARTBEAT.md` 纳入版本库。每次心跳后 `git diff` 能直接看出 Agent 实际做了什么。这比在聊天记录里翻找可靠得多。

## 踩坑点

- **文件膨胀**：Agent 爱写大量“思考过程”。给每个字段限制长度，例如 Signals 每条不超过 140 字，否则一周后文件会变得没法读。
- **自我生成任务**：会出现“优化一下自己”“调研一个新技术”这类无法验收的任务。要求每条任务必须带 measurable result，例如“检查 /health 返回 200，失败则重启服务并记录日志”。
- **并发写冲突**：多个 Agent 或重复触发同时写会互相覆盖。建议单 owner 写，或用 append-only log 加文件锁。
- **忽略外部变化**：Agent 只看 HEARTBEAT.md，可能错过真实系统告警。HEARTBEAT 只做轮询补充，关键事件仍要走 webhook/消息触发。
- **空转**：没有 todo 时，Agent 可能硬找事做。在 prompt 里加规则：如果无 todo，只更新 `last_check` 并退出。

## 可复用建议

- 字段从少开始：`Meta / Top Intent / Tasks / Blockers` 四个足够，跑一周后再加 `Decisions / Signals`。
- 把 HEARTBEAT.md 当“工作台”而不是“档案库”。只保留最近状态，历史交给 git。
- 与 MCP 工具配合：让 Agent 在心跳里调用外部系统状态，而不是只靠文件里的旧信息。例如检查服务健康时直接请求 API，而不是读上一次的记录。
- 人工 review 节奏：每天看一次 git diff，确认 Agent 的主动行为是否符合预期。发现跑偏就改 prompt 或停掉 cron。

## 总结

HEARTBEAT.md 的价值不在于让 Agent 变聪明，而在于给它一个稳定、可轮询、可审计的触发面。对 OpenClaw 这类可编程 Agent 来说，它比复杂的记忆系统更轻，也更容易停用：删掉文件或停掉 cron 就结束。

主动不等于失控。关键是给 Agent 一个明确的“心跳”，并把主动行为限制在可验证的小步动作里。这样它才会在你没提问的时候，依然知道下一步该做什么。

---

