---
title: Prompt 工程实战：Cron 任务的 instruction 该怎么写
feedId: 34418
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

OpenClaw 里，Cron 任务越来越多地被写成自然语言 instruction。好处是灵活，坏处是很多人把 instruction 写成了“愿望清单”：

> 每天帮我看看行业动态，重要内容整理后发给我。

这种写法在交互式对话里可能还能用，因为人可以追问、纠正。但 Cron 是无人值守、可重复、可累积副作用的场景。一旦定时执行，任何歧义都会被放大：模型可能选了错误的工具、推送到错误的频道、重复处理旧数据，或者在凌晨三点因为一个含糊的“看看”而自由发挥。

问题不在模型能力，而在 instruction 没有达到可执行规格的粒度。

## 问题

实际观察到的典型失败有三类：

1. **目标泛化**：只有目的，没有边界。“关注动态”“检查更新”“处理消息”，模型不知道检查哪个源、处理哪些字段、做到什么程度算完成。
2. **步骤缺失**：让模型“整理后发送”，但没说整理成什么格式、发送到哪个 channel、失败怎么办。
3. **副作用不可控**：没有去重和幂等设计，Cron 每次运行都重复推送；没有约束工具权限，任务偶尔读写无关文件。

Cron instruction 的目标不是“好用”，而是“稳定、可预期、可恢复”。

## 做法：用最小结构化模板

我后来把 Cron instruction 统一成六个字段，基本能覆盖大部分自动化任务：

```text
GOAL: 一句话说明本次运行要完成什么。
INPUT: 明确输入来源，例如文件路径、RSS URL、数据库表名。
STEPS: 按顺序列出 2-5 个动作，动词必须可验证。
OUTPUT: 成功时输出什么、写到哪里、是否推送。
FAILURE: 失败时记录什么、是否重试、输出什么状态码。
CONSTRAINTS: 禁止做什么，例如不要修改其他文件、不要连续重试。
```

一个 RSS 摘要任务的示例：

```text
定时任务：daily-rss-digest
Schedule: 0 9 * * *

GOAL: 读取 RSS 列表，生成最近 24 小时的新条目摘要。
INPUT:
- data/feeds.json 中的 RSS 地址列表
- data/seen_guids.txt 作为已处理 GUID 记录

STEPS:
1. 逐个请求 feeds.json 中的 RSS 地址。
2. 仅保留发布时间在最近 24 小时内的条目。
3. 与 data/seen_guids.txt 去重，新 GUID 追加写入该文件。
4. 如果没有新条目，直接输出 "NO_NEW_ITEMS"，不推送。
5. 如果有新条目，按 Markdown 模板生成摘要，保存到 data/digest_YYYYMMDD.md。

OUTPUT:
- 保存摘要文件到 data/digest_YYYYMMDD.md。
- 调用 send_channel 推送到 #rss-digest。
- 控制台只输出状态码：OK / NO_NEW_ITEMS / ERROR。

FAILURE:
- 单步失败时，将错误写入 logs/daily-rss-digest.log。
- 不重试超过 2 次。
- 输出 "ERROR"，不执行后续推送。

CONSTRAINTS:
- 不要读取或修改 data/ 以外的文件。
- 不要生成解释性长文本。
- 不要在没有新条目的情况下调用 send_channel。
```

这个 instruction 的关键不是长，而是每个动作都能被验证。模型跑完后，你知道它成功与否，也知道它改了什么。

## 踩坑点

1. **时区不明确**  
   `Schedule: 0 9 * * *` 在 OpenClaw 中一般有默认时区，但 instruction 里如果写“每天上午九点”，模型并不负责解释时区。配置 Cron 时就把时区定死，instruction 里避免再出现自然时间描述。

2. **任务重叠运行**  
   如果任务执行时间超过 Cron 间隔，可能上一次还没跑完，下一次又启动。对文件写入、推送通知这类任务尤其危险。尽量设计成可覆盖的幂等文件，或者在任务开始前检查锁文件。

3. **去重靠模型“记住”**  
   模型上下文会丢失，Cron 任务不能依赖“上次处理到哪了”。把去重状态外置到文件、数据库或 KV，例如上面的 `seen_guids.txt`。

4. **失败静默**  
   只写“失败就重试”没有用。失败必须落日志，并输出稳定状态码。日志文件最好按日期或任务名区分，避免多个 Cron 任务写同一个文件互相污染。

5. **输出通道过度自由**  
   如果 instruction 只说“发给我”，模型可能选择 DM、群聊、邮件或干脆输出到控制台。Cron 任务的输出通道必须在 instruction 中写死，最好连消息模板也给出。

6. **工具权限过宽**  
   不要让每个 Cron 任务都能访问所有 MCP 工具或插件。只给任务需要的工具权限。权限越窄，误操作范围越小。

## 可复用建议

- **把 instruction 当测试用例写**：先问一句“如果这个任务连续跑 7 天，会产生什么副作用？”能回答出来再上线。
- **先 dry-run，不推送**：先让任务只写文件、不调用通知工具，观察两轮输出，再打开真实推送。
- **状态外置，不依赖上下文**：任何需要跨运行保留的状态都落到文件或数据库。
- **固定输出状态码**：`OK`、`NO_NEW_ITEMS`、`ERROR` 这类极简状态码比自然语言总结更适合定时任务。
- **版本化 instruction**：Cron 任务改 instruction 时留一份旧版本，方便回滚和对比行为变化。
- **限制工具列表**：在任务配置里显式声明可用工具，不要继承全局全量工具。

## 总结

Cron 任务的 instruction 不是写给模型看的“需求描述”，而是写给无人值守系统看的“执行规格”。目标要具体、步骤要可验证、失败要可恢复、副作用要可控制。把自然语言当成接口来设计，才能让 Agent 的定时任务从“偶尔能用”变成“可以长期跑”。

最直接的判断标准是：当你把 instruction 扔给一个不熟悉背景的人，他也能按步骤完成，那这个 Cron instruction 才算合格。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/fc6c0ded8b795be5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/5eb0752642cc5dc2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/e8f5198cdf648d65.png)

