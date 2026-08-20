---
title: Prompt 工程实战：Cron 任务指令怎么写才不翻车
feedId: 33968
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw / Agent / MCP 的自动化里，Cron 任务是最容易“看起来能用，跑起来翻车”的场景。因为定时触发时没有人在旁边纠正，也没有对话上下文可以澄清需求。Agent 只能依靠你写下的 instruction 一次做对。

很多人的 Cron 指令是这样写的：

> “每天早上帮我检查一下服务状态，有问题告诉我。”

交互式聊天里这句话可能没问题，但放进 Cron 就会出问题：Agent 自由发挥、输出格式漂移、工具报错被吞、重复执行、失败也不吭声。

## 问题在哪

Cron 任务和交互式任务有本质区别：

1. **没有反馈回路**：你没法补一句“我不是这个意思”。
2. **失败是静默的**：Agent 可能把工具异常解释成“可能网络抖动”，最后回复“已完成”。
3. **输出必须可验证**：否则你无法判断任务是否真的成功。
4. **时区、重试、重复触发**都会影响结果。

所以，Cron 任务的 prompt 不是“提醒”，而是一份无人值守的作业说明书。

## 做法：把 instruction 写成契约

我建议所有 Cron instruction 都固定成七段：Objective、Success、Tools、Steps、Output、Failure、Constraints。

### 1. 先写“完成定义”，而不是任务描述

不要只写“检查服务状态”，要写清楚执行完成后应该产生什么：

- 生成哪个文件？
- 返回什么字段？
- 状态变化是什么？

例如：

> 成功标准：生成 `reports/health-YYYY-MM-DD.md`，并在最终消息中返回状态码、耗时、失败目标列表。

### 2. 限制工具边界

明确只允许使用哪些工具或 MCP。比如：

> 仅允许使用 `fetch`、`read_file`、`write_file`。禁止调用其他 MCP 或发送额外通知。

否则 Agent 可能会“顺手”调用搜索、发消息、写数据库，造成副作用。

### 3. 规定输出契约

默认输出 JSON，或者指定 Markdown 表格字段。字段要写清类型、是否允许为空。

不要只说“输出结果”。要写：

> 输出为 Markdown 表格：`target, status_code, latency_ms, checked_at`。缺失字段用 `null` 表示，不允许省略列。

### 4. 要求 fail loud

这是最关键的一条。很多 Cron 任务失败是因为 Agent 把错误“美化”成正常结果。

必须写：

> 任何请求失败、超时或状态码非 2xx/3xx，立即停止后续步骤，写入 `logs/health-error-YYYY-MM-DD.md`，并在最终消息中明确失败 target 与错误。不要推测，不要补数据，不要将失败描述为成功。

### 5. 设计幂等与去重

Cron 可能因为重启、DST 或平台重试而重复触发。指令里要加幂等约束：

> 执行前先检查当天报告文件是否已存在。若存在且内容完整，直接返回“已执行过”，不重复写入。

或者用时间戳作为输出文件名的一部分，避免覆盖。

### 6. 要求一行摘要

最后让 Agent 输出一行可读摘要：

> 最终消息只输出一行摘要：成功 n 个，失败 m 个，耗时 Xs。

不要让它输出大段解释。

## 一个可复用的模板

```text
你是 OpenClaw 定时巡检任务执行器。

任务：每天 08:30 检查指定 API 健康状态。
成功标准：生成当日报告文件 reports/health-YYYY-MM-DD.md，
并在消息中返回状态码与耗时。

仅允许工具：fetch、read_file、write_file。
时区：Asia/Shanghai。

步骤：
1. 读取 config/health_targets.json。
2. 对每个 target 发起 GET，超时 10s。
3. 若任何请求失败或状态码非 2xx/3xx，立即停止，
   写入 logs/health-error-YYYY-MM-DD.md，
   并在最终消息中明确失败 target 与错误。
4. 若成功，写入 reports/health-YYYY-MM-DD.md，
   内容为 Markdown 表格：target, status_code, latency_ms, checked_at。
5. 最终消息只输出一行摘要：成功 n 个，失败 m 个，耗时 Xs。

不要推测，不要补数据，不要发送额外通知。
```

## 踩坑点

- **“每天”不等于时区明确**：Cron 表达式本身不含时区，必须在 instruction 里指定时区，否则可能按 UTC 跑。
- **“简单写一下”导致格式漂移**：不固定输出结构，Agent 有时给列表，有时给段落，有时省略字段。
- **工具失败被美化**：模型倾向于把异常解释成“可能网络抖动”，导致任务假装成功。必须强制 fail loud。
- **上下文污染**：长期任务读入过多历史文件，既浪费 token 又容易产生幻觉。限制只读必要文件。
- **重复触发**：同一个 Cron 在重启或夏令时切换时可能触发两次，需要文件锁或幂等 key。

## 可复用建议

1. **模板化**：所有 Cron instruction 都按七段结构写，不要即兴发挥。
2. **输出机器可读**：默认 JSON，给人看的再加 Markdown。
3. **先 dry-run**：第一次不要直接上定时，手动触发一次，检查输出和副作用。
4. **把 Cron 当无人值守 operator**：它没有机会向你确认，所以约束要写在前面。

## 总结

Cron 任务的 instruction 不是写得聪明，而是要让失败可见、输出可验证、执行可重复。工程化 prompt 的本质是契约：定义清楚成功、边界和失败行为。做到这几点，定时任务才能真正无人值守。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5dbff0efb5713e09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/f5d32e1e5fd7e0fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/a1dca879796b9455.png)

