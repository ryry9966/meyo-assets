---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 34066
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

定时任务在 Agent 场景里很容易被低估。很多人把 cron 当普通提醒，instruction 随手写“每天早上帮我整理重要邮件”。结果要么输出三行，要么抓错时间范围，要么重复发通知。原因不是模型不行，而是这类任务没有澄清机会：交互式对话可以追问，cron 触发后只能按 instruction 一次跑完。

OpenClaw 的 cron 任务尤其如此。它通常会和 MCP 工具、插件、通知 webhook 一起用，链路比普通脚本长，任何一环模糊都会放大不确定性。

## 问题

我遇到最多的是四类：

1. 目标模糊：写“重要邮件”但不定义重要。
2. 上下文缺失：任务依赖时间范围、目录、环境，但 instruction 里没写。
3. 输出结构不稳定：模型有时给 Markdown，有时给 JSON，下游解析不了。
4. 失败不可观测：工具调用失败后模型可能继续编造成果，或者直接吞掉错误。

这些问题的本质是：cron instruction 不是聊天指令，而是无人值守的运行手册。

## 做法：把 instruction 写成运行手册

我会用固定模板，至少包含：任务目标、触发时间与时区、输入来源、执行步骤、输出格式、约束、失败策略。

示例：

```markdown
# 任务：每日邮件简报
task_id: daily-email-brief
schedule: 0 8 * * *
timezone: Asia/Shanghai
inputs:
  - INBOX 最近 24h 邮件
  - 重点规则 /data/rules/focus.txt
output:
  - /data/brief/YYYY-MM-DD.md
  - webhook: https://example.com/hook
steps:
  1. 读取 /data/state/daily-email-brief.json，若今天已成功执行则直接退出。
  2. 用 MCP 工具 email_search 查询最近 24h 未读邮件。
  3. 按 /data/rules/focus.txt 过滤，只保留匹配关键词或发件人的邮件。
  4. 按固定模板写入输出文件，并发送 webhook。
constraints:
  - 工具调用失败必须立即停止并写错误日志，禁止根据记忆编造。
  - 单次执行不超过 600s。
  - 不得修改 /data 以外的文件。
failure:
  - 写日志 /var/log/agent-cron/daily-email-brief.log
  - 发送失败告警 webhook
```

其中第 1 步是幂等；第 3 步把“重要”规则外置；第 4 步固定输出；constraints 防止模型乱跑。这个模板可以直接套用。

## 踩坑点

- **把判断全交给模型**：与其让模型自由判断“重要邮件”，不如让工具和规则先过滤，模型只做汇总。判断逻辑前移，稳定性会高很多。
- **客户端时区和服务器不一致**：OpenClaw 容器常用 UTC，但业务可能按 Asia/Shanghai。instruction 里必须写明 timezone。
- **没有幂等设计**：手动补跑会重复发通知。用日期标记、task_id 或状态文件去重。
- **失败被吞**：模型在工具失败后容易“圆场”。指令中要明确“失败立即返回错误，不要继续”。
- **输出格式太随意**：要求 JSON 或固定 Markdown 字段，并给出一个示例；不要只写“输出总结”。
- **把密钥/环境变量硬编码**：secret 通过环境变量或 MCP 配置注入，不要写进 instruction。

## 可复用建议

1. 将 instruction 纳入版本控制，修改要做 diff 和 review。
2. 上线前先用手动入口跑 5 次，检查输出结构、副作用、工具调用是否一致。
3. 加 dry-run 参数，先只看结果不落盘、不发通知。
4. 给任务配错误日志和心跳，连续失败必须告警。
5. 把变量和规则外置到文件，instruction 只做编排，不维护大量配置。

## 总结

Cron 任务的 instruction 不是写得越“聪明”越好，而是越“确定”越好。无人值守环境下，模型自由度越高越危险。把目标、输入、输出、失败策略写清楚，把判断逻辑放到工具和规则里，cron 任务才能稳定跑几个月而不用半夜救火。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/92533e27ee6bf12a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/6713046052b68feb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ae10cb574f5a393c.png)

