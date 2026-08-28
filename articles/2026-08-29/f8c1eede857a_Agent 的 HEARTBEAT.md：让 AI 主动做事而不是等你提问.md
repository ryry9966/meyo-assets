---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 35141
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

OpenClaw / MCP / 插件生态里的 Agent，大多数时候仍然是一个“应答器”。你发消息，它执行；你不开口，它就停在那里。很多自动化场景其实需要 Agent 主动干活：每 15 分钟看一次队列积压，每天早上检查依赖更新，异常时自己通知到群，而不是等运维发现。

常见的做法是外部 cron 定时塞 prompt。但这会带来新问题：Agent 每次被叫醒都像失忆一样从零开始，状态不连续；任务重叠时可能重复执行；规则散落在脚本里，很难 review；通知不是太少就是刷屏。

我们需要一个更工程化的方案：让 Agent 拥有一份“心跳协议”，每次主动醒来时按协议检查、决策、执行、记录。我把这份文件叫 `HEARTBEAT.md`。

## 问题

单纯依赖外部调度，有几个明显短板：

- **上下文断裂**：每次定时任务都是全新会话，Agent 不知道上次检查了什么。
- **规则与执行耦合**：业务判断写在 cron 脚本或 prompt 模板里，改规则要改调度逻辑。
- **缺乏状态持久化**：没有统一记录，出现重复执行或漏报时很难排查。
- **通知不可控**：要么每个周期都发消息，要么只在报错时发，缺少“先判断再通知”的中间层。

这些问题的本质是：Agent 没有自己的“心跳节律”，只是被外部时钟一次次叫起来做一次性动作。

## 做法/步骤

### 1. 建立 HEARTBEAT.md 作为协议文件

把 `HEARTBEAT.md` 放在项目根目录或 Agent 工作区里。内容尽量结构化，不要写长段自然语言，否则 Agent 容易脑补。

一个最小示例：

```markdown
# HEARTBEAT

interval: 15m
jitter: 2m
state_file: .agent/heartbeat.json
quiet_hours: "23:00-07:00"
max_actions_per_beat: 1

tasks:
  - id: queue_depth
    when: every 15m
    action: 通过 MCP 检查队列深度，若超过 1000 则通知 #ops
  - id: outdated_deps
    when: every 6h
    action: 运行依赖检查，仅在新 major 版本出现时通知
```

关键点：`when`、`action`、`state_file` 都是明确字段，不是“适当时候检查一下”。

### 2. 让 OpenClaw 按协议触发心跳

如果你的 OpenClaw 版本支持插件，可以封装一个 heartbeat 插件，读取 `HEARTBEAT.md` 并执行。如果不支持，也可以用系统 cron 调用 OpenClaw CLI 或 HTTP 接口，发送固定指令如 `RUN_HEARTBEAT`。

重要的是：**触发指令只负责唤醒 Agent，真正的业务判断交给 HEARTBEAT.md**。这样规则和调度解耦，改心跳行为不需要动 cron。

### 3. 持久化状态

每次心跳结束，更新 `.agent/heartbeat.json`，记录：

```json
{
  "last_run": "2025-01-01T10:15:00Z",
  "last_task_id": "queue_depth",
  "last_error": null,
  "last_notification_at": "2025-01-01T09:50:00Z"
}
```

下一次心跳先读状态，再决定是否执行、是否需要通知。这样即使 Agent 失忆，状态文件也能帮它恢复判断。

### 4. 主动通知要有冷却和聚合

只有当条件满足时才调用通知 MCP。例如队列深度连续两次超过阈值才告警，或者把 15 分钟内的多个低优先级事件合并成一条摘要。不要每个周期都发“一切正常”。

### 5. 审计日志

每次心跳追加一行 JSONL：

```json
{"time":"2025-01-01T10:15:00Z","task":"queue_depth","decision":"notify","reason":"depth=1200","duration_ms":820}
```

这能让你清楚地知道 Agent 在没人的时候到底做了什么、为什么这么做。

## 踩坑点

- **频率过高导致 token 消耗大**：每 15 分钟一次心跳，如果每次读取大段上下文和全量状态，成本和延迟会明显上升。只读必要的状态摘要，不要把历史日志全塞进去。
- **上下文膨胀**：不要把所有历史心跳记录都保留在会话里。只保留最近几次关键状态，其余放外部日志。
- **并发心跳**：两个心跳同时触发会重复执行。用 lockfile 或检查 `last_run` 时间戳，超过 interval 才能继续。
- **自然语言规则不可靠**：模糊指令会让 Agent 乱执行。把触发条件写成可判定的字段，如阈值、时间窗口、冷却时间。
- **通知噪音**：主动通知太多会被忽略，甚至被静音。设置 quiet hours、冷却时间和聚合策略。
- **权限边界**：主动执行比被动执行更危险。写操作要限制在白名单工具内，删除、发布类操作必须人工确认。

## 可复用建议

1. 从一个小任务开始：先只巡检一个目录或一个 API，跑稳后再扩展。
2. 状态文件用 JSON Schema 校验，防止 Agent 写坏结构。
3. 增加 dry-run 模式：先让 Agent 输出决策但不执行，观察几天。
4. 把 `HEARTBEAT.md` 纳入版本控制，规则变更可以 review 和回滚。
5. 心跳是发现和初判，不是最终告警。重要事件仍建议接入现有告警/值班系统。

## 总结

`HEARTBEAT.md` 不是一份更长的 prompt，而是一份让 Agent 从“被动应答”转向“受控主动”的工程协议。它把周期性任务的规则、状态、通知、审计拆开，使每一次主动行为都可解释、可复现、可回滚。

对 OpenClaw / MCP / 自动化实践者来说，真正值得投入的往往不是更复杂的模型，而是让 Agent 在没人的时候，按规矩做事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/cacee98a5cd8450e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9a07cb769de8679e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0be9ca572f8a7b51.png)

