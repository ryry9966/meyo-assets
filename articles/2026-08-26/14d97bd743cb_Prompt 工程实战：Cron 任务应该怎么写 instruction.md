---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 34788
source: 综合讨论
publishedAt: 2026-08-26
---

在 OpenClaw/Agent 环境里，Cron 任务经常被当成“定时提醒”来写，但实际上它是一段周期性自动执行的 agent 工作流。很多人写 instruction 时只写意图，不写执行契约，结果任务能跑但输出不稳定。结合在插件和 MCP 链路里踩过的几个坑，聊聊 Cron 任务 instruction 的写法。

## 背景

OpenClaw 的定时任务不只是到点发消息，它会触发 agent 按 instruction 执行，读写工具、调用 MCP、产出文件或消息。这个场景下，instruction 就是 agent 与外部系统之间的接口说明。如果写得太口语化，agent 会用默认行为补全，补出来的部分往往不可控。

## 常见问题

- 目标模糊：“每天帮我整理一下数据”——不知道来源、范围、输出位置。
- 失败路径缺失：接口超时、数据为空、权限不足时，agent 自行决定，可能重试或跳过。
- 时区和幂等没写：UTC 与本地时间混淆，重复执行产生重复记录。
- 输出格式不稳定：同一任务今天给 JSON，明天给 Markdown，下游 MCP 解析失败。

## 做法 / 步骤

我现在用固定模板，重点是把 instruction 写成执行规范。

1. **Objective**：一句说清任务目标
2. **Trigger**：时间、时区、频率
3. **Inputs**：数据来源、工具/MCP 名称、参数
4. **Steps**：按顺序列出步骤，用 If/Then 处理分支
5. **Output**：指定格式、目标位置、示例
6. **Failure**：失败、重试、跳过策略
7. **Constraints**：禁止事项、幂等条件

示例片段：

```
Objective: 每天汇总销售数据并写入 report 表
Trigger: 每天 08:00 Asia/Shanghai
Inputs: MCP tool get_sales_data(date=yesterday)
Steps:
  1. 调用 get_sales_data 获取昨日数据
  2. 如果数据为空，返回 SKIP，不写入
  3. 按 region 聚合 amount
  4. 生成 JSON 数组，字段固定为 ["region","amount"]
Output:
  写入 database.weekly_report，使用 upsert，去重键为 (date,region)
Failure:
  接口超时重试 1 次，仍失败则记录日志并结束，不要尝试其他工具
Constraints:
  不要修改历史数据；不要自行推测缺失字段
```

关键不是写得多长，而是每个字段都可验证。

## 踩坑点

1. 不要只用“帮我整理”这种意图句，至少要带数据源和输出位置。
2. 时区要写死，不要写“早上”，别让 agent 猜。
3. 重复任务必须定义去重键或 upsert，否则一跑就脏数据。
4. 共享会话会带入历史上下文，cron 指令最好用独立会话或每次清空相关记忆。
5. 输出给 MCP/插件时，明确 JSON schema，别用“合适格式”这种词。
6. 先让 agent 跑 dry-run 模式，只打印计划不写数据，确认无误后再放开。

## 可复用建议

- 维护 cron prompt 模板，把 Objective / Trigger / Inputs / Steps / Output / Failure / Constraints 拆成参数。
- 指令里多用 If/Then 和 checklist，少用“聪明地”“合理地”这类模糊词。
- 英文技术名词保持英文，中文说明可以辅助，不要中英混用搞乱字段。
- 每改一版 instruction，保留版本号，记录一次运行结果，方便回滚。
- 稳定优先于聪明：让 agent 只做确定的事，范围之外宁可返回 SKIP 也不要自由发挥。

## 总结

Cron 任务 instruction 的本质，是把“周期性意图”翻译成“可验证的执行规范”。如果定时任务经常跑偏，先别调模型参数，看看 instruction 有没有把时区、输入输出、失败策略和幂等条件写清楚。通常补上这些，比换模型或加更多上下文更有效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/5e0a80f855b23ed2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/0d58197d55636d61.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/7321e82f015eab8a.png)

