---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 34438
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景
在 OpenClaw / Agent / MCP 这类自动化环境里，Cron 任务已经从“定时提醒”变成了一种无人值守的自动执行单元。它可能读邮件、抓 RSS、调用工具、写文件、发通知。很多故障不是模型能力不够，而是 instruction 写得太像给人类看的备忘：时间模糊、边界不清、失败不处理、输出随意。

## 问题
常见的 Cron instruction 长这样：

> 每小时帮我看看最新邮件，有重要的总结一下发给我。

这在交互式对话里问题不大，但放进 Cron 后会暴露：
- “每小时”是按 UTC 还是本地时区？
- “最新邮件”要靠什么状态判断哪些已经处理过？
- “重要”的标准是什么？摘要多长？发到哪个频道？
- 邮箱暂时不可用时，是重试、跳过还是报警？
- 任务上下文是否被上一轮对话污染？

无人值守时，歧义会被持续放大：重复处理、漏数据、静默失败、输出格式失控。

## 做法：把 instruction 写成一份小型运行手册
对一个生产可用的 Cron 任务，建议固定以下字段：

| 字段 | 作用 |
|---|---|
| Trigger | 精确的 cron 表达式 + 时区 |
| Objective | 本次运行要完成什么，不要做什么 |
| Input | 数据来源、读取范围、状态标记 |
| Constraints | 权限边界、禁操作、过滤规则 |
| Success Criteria | 一次成功运行必须满足的条件 |
| Failure Policy | 重试次数、退避、报警渠道、降级动作 |
| Output Format | 结构化输出、长度上限、投递目标 |
| Idempotency | 幂等键、状态文件、去重方式 |

下面是一个可参考的示例：每小时读取未读邮件并生成摘要。

```text
[Trigger]
cron: 0 * * * *
timezone: Asia/Shanghai

[Objective]
读取 Gmail 未读邮件，生成中文摘要和待办项。只读，不回复、不删除、不标记已读。

[Input]
inbox 中 UID > last_processed_uid 的邮件；last_processed_uid 存储在 /state/email_cron.json。

[Constraints]
忽略 newsletter 和 promotions 分类；仅保留发件人在 VIP 列表或含 action required 的邮件；
不得将邮件原文直接转发到公共频道。

[Success Criteria]
输出 JSON 数组，每项包含 uid、subject、summary_3_bullets、action_items、priority。
无新邮件时输出 {"new":0}。

[Failure Policy]
Gmail 不可用时重试 1 次，间隔 5 分钟；仍失败则发送错误通知到 #alerts，
并保留 last_processed_uid 不变。禁止猜测数据。

[Output Format]
Markdown 表格，最多 20 行，发送到 #daily-digest。只有 priority=high 时才 @值班人。

[Idempotency]
每次运行先读 state，再处理 uid > last_processed_uid，完成后更新 state。
若中途退出，不得更新 state。
```

关键不是字段数量，而是把“无人值守时会遇到的决策”提前写死：处理范围、失败动作、输出形状、幂等位置。

## 踩坑点
1. **用自然语言描述时间**  
   “每小时”“每天早上”“过一会儿”在不同环境里解析不一致。写明确 crontab 表达式和 IANA 时区，例如 `TZ=Asia/Shanghai`。

2. **依赖历史上下文**  
   Cron 运行可能复用之前的 conversation，导致上一轮未清理的指令、文件名、返回值污染当前任务。应在 instruction 开头声明：本次运行是独立任务，忽略历史上下文，只按当前字段执行。

3. **没有幂等设计**  
   如果任务中断或被重复触发，没有 UID/游标/状态文件，就会重复读、重复发、重复写。对邮件、RSS、消息流任务，必须有一个可持久化的 `last_processed_uid` 或 timestamp。

4. **失败静默**  
   很多 Agent 在工具调用失败后倾向于“友好地”跳过或返回空结果。需要明确 Failure Policy：允许重试几次、失败后写日志、发报警、是否允许返回空。

5. **输出格式过松**  
   “发我一份总结”在定时任务里会变成一段难以消费的自由文本。要求固定 JSON 或 Markdown 表结构，并限制行数/长度，避免通知风暴。

6. **工具权限未收敛**  
   Cron 任务如果带着全套工具权限，可能因为理解偏差执行删除、发送、修改等操作。只授予任务需要的最小工具集，写操作加白名单或二次确认。

## 可复用建议
- 把 Cron instruction 当成 SOP，而不是一句祈使句。
- 先手动触发一次，查看日志和输出是否符合预期，再启用定时。
- 为每个任务单独建状态文件，不要多个任务共用一个游标。
- 输出格式固定 schema，失败输出也要固定，方便下游解析。
- 用 `dry-run` 模式先打印将执行的动作，不真正写数据。
- 定期回看 Cron 任务的执行日志，很多偏差是逐步积累出来的。

## 总结
Cron 任务的 instruction 不是在教 Agent“做什么”，而是在定义一个小型自动化系统。把时间、范围、边界、幂等、失败和输出都写清楚，它才能真正脱离人工值守稳定运行。对 OpenClaw / MCP 这类工具链来说，一条好的 Cron prompt，本质就是把一次人工对话的经验，沉淀成可重复执行的运行手册。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/58e2d976e5551075.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/51412c89a79dfd14.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/211516f3c95d25da.png)

