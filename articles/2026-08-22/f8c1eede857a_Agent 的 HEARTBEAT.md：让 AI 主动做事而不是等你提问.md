---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 34170
source: 综合讨论
publishedAt: 2026-08-22
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

## 背景

大多数 agent 默认是被动的：你提问，它回答；你下指令，它执行。但在监控、巡检、订阅汇总、定时数据整理这类场景里，真正需要的是 agent 能自己判断“现在该做什么”，而不是每次靠人类推一把。

常见做法是用 cron 或 OpenClaw 的定时插件唤醒 agent。但问题在于：多数定时任务只带一句固定 prompt，agent 每次醒来都像失忆一样，不知道上次跑到哪、哪些已经处理过、这次是否应该跳过。结果就是重复执行、漏报、上下文断裂，甚至每次定时任务都烧掉大量 token 重复做同一件事。

## 问题在哪

缺少一个跨唤醒周期的外部状态文件。没有状态，agent 的“主动”就无从谈起。你给它再长的 system prompt，它还是只能基于当前输入做一次性反应。

所以我们需要一个很小的锚点文件：`HEARTBEAT.md`。它让 agent 在每次被唤醒时先读心跳，再决定行动。

## 做法：给 agent 一个最小心跳文件

### 1. 建立文件结构

在 agent 工作目录下建一个 `HEARTBEAT.md`，用 frontmatter 存结构化状态，正文放人类可读摘要。字段尽量少：

```markdown
---
last_run: 2026-05-12T09:30:00Z
last_check: 2026-05-12T09:40:00Z
status: ok
error_count: 0
lock: false
max_pending: 5
pending:
  - id: fetch-weather
    action: 抓取目标城市天气并写入 daily-brief
    due: 2026-05-12T10:00:00Z
next_run_hint: 2026-05-12T10:00:00Z
---
# Heartbeat

上次任务：成功抓取 3 个源，写入 daily-brief。
下次建议：10:00 抓取天气。
待办：1 项。
```

### 2. 定时触发只做一件事

cron 或 OpenClaw 定时插件不要写具体业务指令，只固定传一个简短提示：

```
Read HEARTBEAT.md, decide if any action is due, then act.
```

这样 agent 的行为由心跳文件驱动，而不是由定时任务里的硬编码 prompt 驱动。

### 3. 让 agent 按固定循环执行

每次唤醒后，agent 遵循同一个循环：

1. 读取 `HEARTBEAT.md` 的 frontmatter。
2. 判断当前时间是否超过 `next_run_hint` 或某个 pending 的 `due`。
3. 如果没有到期且无异常，只更新 `last_check`，然后退出，不做无谓搜索。
4. 如果有到期任务，先输出一个简短 plan，然后执行。
5. 执行完成后回写心跳：更新 `last_run`、`status`、`error_count`，清理或新增 pending，设置下一次 `next_run_hint`。

如果使用 MCP 的 filesystem server，可以把 `HEARTBEAT.md` 和 `state.json` 都交给 agent 读写，避免依赖模型记忆。

### 4. 保持心跳文件瘦小

详细执行日志不要全塞进 `HEARTBEAT.md`。可以只保留最近 3 次执行摘要，完整上下文写到 `runs/2026-05-12-0930.md`，心跳文件只存指针和状态。

## 踩坑点

- **并发写冲突**：如果两个定时任务重叠触发，可能同时读写心跳。加 `lock: true` 或 `pid` 字段，让 agent 启动时先检查锁；或者用单实例 cron。
- **frontmatter 被写坏**：模型回写 YAML 时可能改出非法格式。每次回写后做简单校验，失败则回退上一版。更稳的做法是用 JSON 存 `state.json`，`HEARTBEAT.md` 只放人类可读摘要。
- **心跳膨胀**：不要无限追加日志。保留最近几条，其余归档，否则很快撑爆上下文窗口。
- **主动过度**：agent 可能不断给自己发明新 pending，导致永远“有事做”。设置 `max_pending`，超过后必须人工 review。
- **时区不一致**：时间戳统一 UTC 或带时区，否则 agent 会误判 due 是否到期。
- **空转消耗**：如果没有待办，明确只更新 `last_check` 就退出，不要读大段上下文或做无关操作。

## 可复用建议

- 字段固定，控制在 10 个以内，减少模型自由发挥。
- 用 Git 管理心跳文件，每次变化可以 `git diff`，排障比翻日志快。
- 把“读心跳 → 出 plan → 执行 → 回写”写进 agent 的系统提示，形成固定节奏。
- 设置连续 `error_count >= 3` 时主动推送通知，而不是等人工发现。
- 首次接入先 dry-run，或把 pending 的 `due` 设到未来，观察几个周期再放开。
- 心跳文件不是日志，它只回答三个问题：上次做到哪、现在该做什么、下次什么时候醒。

## 总结

HEARTBEAT.md 不是复杂方案，它是一个很小的外部状态文件。它的作用是把 agent 的每一次定时唤醒连成一条线，让 agent 从“请求-响应”变成“状态-触发”。

让 AI 主动做事，靠的不是更长的 prompt，而是一个记得住上下文的锚点。对 OpenClaw 这类可以定时唤起 agent 的环境来说，这个锚点比堆功能更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/232f68d1ef5b5c79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7af19f3e19fb822d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b5f0d8b824c1cedc.png)

