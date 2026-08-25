---
title: Cron 任务 instruction 不是备忘，是无人值守运行手册
feedId: 34640
source: 综合讨论
publishedAt: 2026-08-25
---

# Cron 任务 instruction 不是备忘，是无人值守运行手册

## 背景

在 OpenClaw 这类 Agent 自动化环境里，cron 任务最常见的写法是一句自然语言：

> 每小时检查一次服务，挂了就提醒我。

交互式场景下这句话可以成立，因为人可以追问、补充上下文。但定时任务每次都是冷启动，Agent 只能依赖 instruction 和工具返回结果。模糊表达会被 Agent 的默认行为填补，结果时好时坏：有时候判断阈值漂了，有时候输出格式变了，有时候同一异常每小时重复通知。

## 问题

cron 任务和对话 prompt 有几个关键差异：

- 没有追问机会，错了就错了；
- 重复执行，上一次的状态不能靠记忆；
- 无人确认，权限边界要提前锁死；
- 通知需要冷却，否则会变成告警轰炸。

真正难的从来不是“跑一次”，而是跑了第 20 次、第 50 次之后，行为仍然可预期。

## 做法/步骤

不要写备忘，写一份可审计的运行单。核心是把默认行为压缩掉，把状态、阈值、时区、输出、失败处理全部显式化。

### 1. 固定输入与上下文来源

不要写“检查服务是否正常”，要写清楚数据从哪来、怎么判断。

例如：

> 调用 `tools/get_service_status()` 获取服务状态；若 `status != "ok"` 且 `retry_count > 2`，判定为异常。

如果需要历史状态，必须落盘，不能依赖“上次说过”。例如：

> 读取 `./state/last_alert.json` 的 `last_alert_at`，若距当前时间小于 24 小时，则本次不通知。

### 2. 给出固定输出 schema

避免 Agent 自由发挥。交互式输出可以随意，cron 输出需要结构化。

> 输出 JSON：`{"fired": false, "reason": "ok", "notify": false}`  
> 只有 `notify=true` 时才调用通知工具；不要输出 Markdown 报告。

### 3. 时间、时区、窗口要显式

很多 cron 翻车是因为系统用 UTC，用户按本地时间看结果。指令里必须写清：

> 所有时间以 `Asia/Shanghai` 为准；只在 09:00–21:00 发送通知；非窗口期只记录不通知。

### 4. 定义失败语义

没有失败策略的 cron，要么静默失败，要么重复轰炸。要写清楚：

> 工具超时 8 秒视为失败；失败写入 `logs/cron_error.log`；连续失败 3 次才通知人工。

### 5. 最小可复用模板

```text
Role: 定时巡检执行器
Schedule: 每 30 分钟，Asia/Shanghai
Inputs:
- 状态来源: tools/get_service_status()
- 状态文件: ./state/last_alert.json
- 通知渠道: tools/notify(webhook_url)
Steps:
1. 获取当前时间，记为 checked_at。
2. 调用 get_service_status()，超时 8s，失败重试 1 次。
3. 若 status != "ok" 且 retry_count > 2，判定为异常。
4. 读 last_alert.json 的 last_alert_at；若距 now < 24h，则 notify=false。
5. 需要通知时，仅发送 service_name、status、retry_count、checked_at。
6. 更新 last_alert.json。
Output:
- 追加写 logs/cron_service_check.jsonl，每行一条 JSON。
- 不输出 Markdown 报告。
Failure:
- 工具失败写 logs/cron_error.log；连续失败 3 次才通知。
Constraints:
- 只读状态、写指定状态文件、发指定通知；不得修改服务配置。
```

## 踩坑点

- **把示例当规范**：只给“像这样输出”，没有字段定义，Agent 会不断变形。
- **依赖对话记忆**：cron 冷启动，上一次的机器名、路径、结论都不存在，必须落到文件或参数。
- **不显式时区**：任务按 UTC 执行，用户按本地时间理解，告警时间对不上。
- **通知没有冷却**：异常持续 6 小时，就发 12 条通知。需要 24 小时去重或状态位。
- **工具权限给太宽**：定时任务无人确认，只给读状态、写指定状态文件、发指定通知；禁止删除、发布、改生产配置。
- **输出被截断**：长任务要求“总结一下”，Agent 可能输出冗长内容。明确输出长度和字段，避免后续解析失败。

## 可复用建议

- 把 instruction 保存成文件，纳入版本管理。修改可 diff，回滚可追踪。
- 手动触发至少测试三种情况：正常、异常需要通知、异常但在冷却期内。
- 状态文件用 JSON，不要用自然语言记录。自然语言状态无法可靠解析。
- 关键 cron 加自检：运行前确认当前时间、状态文件可读写、通知渠道可达。

## 总结

Cron 任务的 instruction 不是“告诉 Agent 要做什么”，而是给一个无人值守执行器一份可审计的运行手册。把状态、阈值、时区、输出、失败处理写清楚之后，正常路径应该是无聊地重复，异常路径应该可追踪、可收敛。能用字段定义解决的问题，不要留给 Agent 临场判断。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/83d12e11088c4f3c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/7167fc4bd3df6ec9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/4e58c09c5808e5d8.png)

