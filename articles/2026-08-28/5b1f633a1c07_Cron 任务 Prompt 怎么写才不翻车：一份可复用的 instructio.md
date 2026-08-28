---
title: Cron 任务 Prompt 怎么写才不翻车：一份可复用的 instruction 模板
feedId: 35009
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 这类 agent 自动化环境里，cron 任务很容易被当成“定时提醒”。例如到点让 agent 发一句话、跑一个固定命令。但一旦希望 agent 做判断——比如每天汇总未关闭 issue、每小时检查异常日志、每周生成周报——真正决定稳定性的往往不是模型能力，而是 instruction 写得是否足够“无歧义”。

定时任务和对话式任务最大的区别是：没有二次澄清机会。你写错了，它会在你睡觉时认真执行错。

## 常见问题

- 指令过短，模型自由发挥，输出格式每次不同。
- 上下文缺失，只给任务目标，不给数据来源、工具路径或执行边界。
- 失败静默，任务报错但无人知道。
- 输出没有契约，下游无法消费。
- 时间表述模糊，导致重复处理或漏处理。

## 一个可复用的 instruction 结构

我建议把 cron instruction 固定成六段：Goal / Inputs / Steps / Output Contract / Failure Handling / Constraints。示例：

```yaml
schedule: "0 9 * * *"
timezone: "Asia/Shanghai"
instruction: |
  Goal: Summarize open high-priority issues in the backend project.
  Inputs:
    - Use the GitHub tool.
    - Repo: org/backend
    - Query: label=high, state=open
  Steps:
    1. Fetch issues matching the query.
    2. Only include issues updated in the last 24 hours.
    3. Group by owner.
    4. Format result using the output template.
  Output Contract:
    - Send one message to #ops-daily.
    - Use this exact template:
      ## Daily Issue Summary
      - Total: {n}
      - Owner: {name} -> {count}
    - If total is 0, send exactly: "No high-priority issues."
  Failure Handling:
    - If GitHub tool fails, send an error message to #ops-alert.
    - Do not fabricate issue titles or counts.
  Constraints:
    - Read-only access.
    - Do not edit or close any issue.
    - Do not mention users by real name; use GitHub handle.
```

这个示例看起来很长，但大部分内容是在消灭自由度。定时任务能稳定运行，通常不是因为提示词写得聪明，而是因为没给模型太多临场发挥的空间。

## 踩坑点

1. **时间窗口模糊**  
写“昨天”很容易出问题。执行延迟、时区、夏令时都可能让“昨天”偏移。应写“上一个 24h 窗口：执行时间向前推 24 小时”，并在配置里固定 timezone。

2. **重复执行不幂等**  
cron 重试或调度抖动会导致同一时间窗口被处理两次。可以在 instruction 里加：如果最近一次日志显示该窗口已处理，则跳过。或者在输出前检查目标频道是否已有相同 summary。

3. **失败静默**  
模型可能“礼貌地”不输出错误。要明确：工具失败必须报告，不允许用空结果替代错误。

4. **把大量背景塞进 instruction**  
定时任务不是知识库。长上下文会让模型忽略关键约束。把 repo 列表、账号映射、发送模板放到文件或 MCP 工具里，instruction 只写“从 X 读取列表”。

5. **权限过宽**  
定时任务无人值守，尽量只读。即使需要写操作，也要明确白名单：只能更新哪个表、只能发哪个频道。

## 可复用建议

- 写一个 dry-run 参数：先手动执行一遍，看输出是否稳定。
- 固定输出模板：给正则或示例，别让 agent 自己设计 markdown。
- 把 instruction 参数化：日期、repo、频道都作为变量注入。
- 失败路径单独通知：正常输出到业务频道，错误输出到告警频道。
- 每季度回看一次失败记录，把新的歧义补进模板。

## 总结

Cron 任务的 instruction 不是“写得更详细”，而是“写得更确定”。把目标、输入、步骤、输出、失败、边界六个问题回答清楚，比堆一堆“请认真、请准确”有用得多。对无人值守任务来说，可预测比聪明更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/a4224b470c53cdac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/4fd4c9a71d5551cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/bb30261f209a85a6.png)

