---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 34941
source: 综合讨论
publishedAt: 2026-08-27
---

在 OpenClaw 里把定时任务交给 Agent，最容易被忽略的不是 cron 表达式，而是 instruction 的边界。很多翻车现场都有一个共同点：开发者写的是“每天帮我处理一下收件箱”，Agent 却在错误的时间、错误的目录、以错误的方式执行，甚至重复写文件。本文整理一套可落地的 Cron instruction 写法。

## 背景：为什么 Cron 任务的 prompt 更容易失控

定时任务没有实时对话兜底。执行时你不在场，Agent 必须依赖 instruction 里预置的判断完成“触发、读数据、做处理、写结果、报错”这一整条链路。只要某个环节描述含糊，它就会用模型的默认倾向补齐，补出来的往往不是工程上想要的行为。

常见问题包括：时区理解不一致、频率用自然语言描述导致漂移、任务范围过宽、只写目标不写禁止项、失败后无记录、重复执行不幂等。

## 问题：Cron instruction 需要明确哪些信息

一个可执行的 Cron instruction 至少要覆盖七件事：

1. 触发条件：cron 表达式和时区。
2. 任务目标：一句话说明完成后的可验证结果。
3. 作用范围：允许读写哪些目录、文件或 MCP 工具。
4. 禁止行为：明确不允许修改、删除、外发或覆盖什么。
5. 输出方式：写入哪里，格式是什么。
6. 无任务时的行为：是输出 NOOP 还是终止。
7. 失败处理：重试次数、日志位置、是否通知。

注意这些信息应写在 instruction 内或外层的系统字段里，不要让模型从 cron 表达式中反推任务内容。

## 做法：一个可复用模板

下面是一个偏自动化场景的 instruction 模板，可直接改成自己的任务：

```text
You are a cron job executor.

Schedule: 0 9 * * 1-5, timezone Asia/Shanghai.
Goal: read new items under /data/inbox, produce a concise summary in Chinese.

Scope:
- Read-only: /data/inbox
- Write-only: /data/out/daily-summary.md
- Allowed tools: local_file_read, local_file_append
- Never delete, move, or overwrite source files.

Output contract:
- Append summary to /data/out/daily-summary.md, starting with a date heading.
- Return JSON only: {"status": "ok|noop|error", "processed": 0, "output_file": "", "error": ""}

No new items:
- Return {"status":"noop","processed":0,"output_file":"","error":""} and do not touch output file.

Failure:
- Retry once after 10 minutes.
- If still failing, append error to /data/log/cron-error.log and return error JSON.

Safety:
- Before any write, print the planned content as a diff-style patch.
- If uncertain about scope, return error instead of guessing.
```

这个模板的关键不是“写得长”，而是把决策点提前固定：时区、目录、工具、输出格式、无任务行为、失败处理、写前确认。模型在执行时只需要做“查找新项—生成摘要—按格式追加”，没有太多自由发挥空间。

## 踩坑点

**时区不写等于随机。** 服务器可能 UTC，OpenClaw 运行时可能容器时区不同。写上 `Asia/Shanghai` 比写“本地时间”可靠。

**频率用自然语言容易漂移。** “每天早上跑一下”“隔一会儿看看”会被模型理解成不同粒度。给 cron 表达式，必要时写清楚“不要自行调整频率”。

**一个任务塞进多个目标。** “清理文件、生成报告、发通知”最好拆成多个 cron，或分步骤但每一步有明确前置条件。混合目标会让失败定位困难。

**只写要做什么，不写不能做什么。** Agent 可能为了完成任务而删除临时文件、覆盖历史输出、调用无关 MCP 工具。把禁止项明确写出来，能降低这类风险。

**忽略幂等。** 如果运行两次会重复写入，就需要用日期 heading、游标文件、状态文件或 `processed` 标记来去重。否则一个重试就可能污染数据。

**失败静默。** Cron 场景没有人实时盯。要求返回 JSON，并在外层把非 `ok` 状态接入日志或通知。否则任务失败了三天都没人发现。

## 可复用建议

把 Cron instruction 当成“运行手册”而不是“愿望清单”。建议固定以下习惯：

- 触发与执行分离：外层配置 cron 和 timezone，instruction 只写执行契约。
- 始终要求结构化返回：JSON 比自然语言总结更容易监控。
- 给默认安全路径：条件不满足就 NOOP，不确定就返回 error。
- 先 dry-run：在 instruction 里要求“先列出将要处理的文件，不要修改”，确认行为后再放开写权限。
- 保留最近运行状态：每次执行记录 run_id、cursor 或时间戳，避免重复处理。
- 少用形容词：把“智能处理”“合理整理”换成可验证的输入、输出和禁止项。

例如，与其写“每天整理收件箱并归档”，不如写：“读取 `/data/inbox` 中最近 72 小时内新增的 `.md` 文件，按日期归档到 `/data/archive/YYYY-MM-DD/`，只复制不删除，完成后返回 JSON。”

## 总结

Cron 任务的 instruction 本质上是在替不在场的你设置护栏。把触发、时区、范围、输出、无任务行为、失败处理和幂等要求写清楚，比堆叠“请智能执行”更有效。好的定时任务 prompt 不是表达意图更强，而是让错误路径更少。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/20aaed1ce09c2e41.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/089f602b345e9acf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/3fbc25069658a1b0.png)

