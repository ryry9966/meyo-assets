---
title: Cron 任务的 instruction 不是提醒，是无人值守 runbook
feedId: 34578
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw / Agent / MCP / 插件自动化的场景里，Cron 任务经常被当成“定时提醒”来用。但实际上，它是在没有用户实时干预的情况下，让模型读取指令、调用工具、产生副作用。

一个提醒写不好，最多烦一下；一个 Cron instruction 写不好，可能重复发消息、误改配置、泄露密钥，或者在凌晨把 token 烧光。

## 问题

最常见的 Cron instruction 长这样：

```text
每天早上 8 点帮我看看新闻，有重要的发我。
```

这句话在执行时至少要猜：哪个时区？新闻源是什么？什么叫“重要”？发到哪个渠道？失败怎么办？是否允许执行 shell？是否允许读其他文件？

没有人在循环里纠正，模型只能靠概率补齐。于是同一个任务连续跑三天，可能三种行为。

## 做法：把 instruction 写成 runbook

我建议把每个 Cron 任务当成“无人值守的操作手册”，至少包含这些字段：Role、Trigger、Input、Steps、Output、Error handling、Constraints。

一个更可用的例子：

```text
[Role]
你是定时执行助手，只执行本任务，不扩展目标。

[Trigger]
每天 08:00 Asia/Shanghai 执行；若错过，不补跑。

[Input]
读取环境变量 NEWS_API_KEY，请求 /v2/top-headlines?country=cn&category=technology。
超时 10 秒，失败重试 1 次。

[Steps]
1. 请求上述接口。
2. 过滤标题包含“AI / 开源 / 自动化”的条目。
3. 最多保留 5 条，按发布时间倒序。
4. 每条生成不超过 80 字摘要，并附原始链接。

[Output]
发送到 #daily-news 频道，格式：
- 标题
- 摘要
- 链接
不发送额外解释。

[Error handling]
接口失败或无匹配时，只记录日志，不发送空消息。

[Constraints]
- 只读，不修改配置。
- 不执行 shell 命令。
- 不读取其他文件。
- 使用当天日期做去重 key，避免重复发送。
```

这版没有复杂的 prompt 技巧，核心是把“隐式上下文”补全。

## 踩坑点

1. **时区不写清楚**。写“早上 8 点”时，调度器可能用 UTC。统一写 IANA 时区，例如 Asia/Shanghai。
2. **权限没有收窄**。Cron 指令如果没有约束，可能调用删除、发送、写入配置等工具。显式写“只读”“禁止执行 shell”可以减少事故。
3. **失败静默**。一定要在指令里规定失败时的行为，哪怕只是记录日志或发一条低优先级告警。
4. **输出渠道太模糊**。“发给我”不够，应该指定 channel、设备或 MCP 工具名。
5. **密钥写进 instruction**。把 token 直接写在指令里，等于把凭证放进调度任务和日志。用环境变量或 secret 引用。
6. **没有 dry-run**。先手动触发一次，或者加一个 dry-run 参数，确认输出格式和副作用符合预期。

## 可复用建议

- 使用同一套模板：Trigger / Input / Steps / Output / Error handling / Constraints。
- 把 Cron instruction 纳入版本管理，改动像代码一样 review。
- 如果任务用 MCP 工具，直接写工具名和参数 schema，不要用自然语言描述“去调用那个发消息的工具”。
- 给任务加幂等 key，防止重复执行造成重复发送。
- 前几次运行先发草稿或只记录日志，稳定后再开正式发送。
- 定期清理旧任务。很多事故来自“当时测试用的 cron 忘了删”。

## 总结

Cron 任务的 instruction 不应该是一句意图，而应该是给一个无监督 Agent 的操作手册。把时区、输入、步骤、输出、异常和边界都写清楚，它才会像一个稳定的小型自动化脚本，而不是每天随机发挥的模型。

稳定不是靠更复杂的 prompt，而是靠更完整的上下文和更窄的权限。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d1d0da55fc9dbb98.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f3d6f576ee83446b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/666968f13d30e139.png)

