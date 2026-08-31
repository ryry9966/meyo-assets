---
title: 让 Agent 从“问答机”变成“值班员”：proactive 能力的分层落地
feedId: 35631
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

多数 Agent 和 MCP 插件仍以 turn-based 为主：你问一句，它动一次。但在巡检、依赖更新、异常检测、日历提醒这类场景里，真正有价值的是“不等你开口就把事情办了”。Proactive 不等于让模型更爱说话，而是建立一个事件驱动的控制回路：来源感知、判断是否值得动、动作受控、结果回流。

## 问题

直接把 cron 或 webhook 接给 Agent，很容易做成两类失败：一是“通知轰炸器”，每 5 分钟推送一次“系统正常”；二是“自动乱动器”，低置信度建议直接执行，出了事只能翻日志。更麻烦的是重复触发、上下文过载、权限扩大后无法降级。

## 做法：把 proactive 拆成四层

在 OpenClaw 里，我把它拆成 Event Source、Evaluator、Action Gate、Audit/Feedback 四层。

**1. Event Source：事件统一收口**

来源可以是 MCP resource subscription、webhook、cron、file watcher。每个事件固定 envelope：`source`、`entity`、`metric`、`value`、`fingerprint`、`timestamp`、`ttl`。不要直接把 raw log 塞给模型。统一事件结构后面去重和审计都会容易很多。

**2. Evaluator：先判断值不值得动**

先用阈值、去重窗口、跨源条件、静默期做第一层过滤；再让 Agent 生成“建议动作 + 证据 + 置信度”。低置信度或证据不足时只记录，不进入行动层。这个阶段的目标是压低噪声，而不是展示模型有多聪明。

**3. Action Gate：动作必须过闸**

先只允许 read-only 和通知类动作；命令 allowlist + 人工确认。首次上线至少跑 48 小时 shadow mode：只记录预测动作，不执行。动作要求幂等、有超时、可回滚。没有这些约束，proactive 很容易从省心变成添乱。

**4. Audit/Feedback：让系统能纠错**

用 append-only 日志保存 `event → decision → action → result`。把人工反馈“有用/噪音/错误”回收为规则或阈值，而不是只靠模型记忆。没有反馈回路的 proactive，通常会在第二周被用户关掉。

## 踩坑点

- **高频 push 最致命**：用户很快会静音。要有 batching、quiet hours、每日上限。
- **MCP server 没给幂等键或 TTL**：同一条 webhook 会重复触发。必须用 `fingerprint + ttl` 去重。
- **上下文过载**：不要把完整对话塞进每次 proactive 判断，只保留相关实体的最近状态。
- **模型循环触发**：action 自身产生新事件，引发死循环。行动后加冷却时间，并标记来源。
- **权限隐形扩大**：关闭人工确认后没有降级路径，误执行只能复盘。保留“只通知模式”开关。

## 可复用建议

- 先跑 shadow mode，收集 3–7 天真实数据，再逐步放开通知、只读动作、写动作。
- 每个 proactive 卡片至少包含：证据、建议动作、有效期、静音/纠错入口。
- 所有动作从 allowlist 开始，不在 allowlist 的命令只能生成建议，不能直接执行。
- 每个事件源设置配额：每小时最多 N 次决策、M 次动作。
- 把 proactive 能力封装成独立 plugin/MCP tool，便于停止、替换和复用。

## 总结

Proactive 不该是模型自由发挥，而是一个受控的事件驱动闭环：来源标准化 → 低噪声评估 → 动作闸门 → 审计反馈。工程上先证明“预测有价值”，再放开“自动执行”。这样它才能从玩具变成真正省心的值班员。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8b734147d5a6ad39.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/bf3067526d3b9339.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/5feefa9b5f35269c.png)

