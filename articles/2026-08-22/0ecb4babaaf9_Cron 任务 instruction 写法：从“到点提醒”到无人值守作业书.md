---
title: Cron 任务 instruction 写法：从“到点提醒”到无人值守作业书
feedId: 34217
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 的自动化实践中，Cron 常用于日报、巡检、抓取、备份等定时任务。但很多人直接把交互式提示塞进 Cron，比如“帮我看看今天有什么重要消息”。定时触发后，结果往往不可控：有时闲聊、有时漏步骤、有时重复执行。核心原因是：Cron 没有澄清机会，Agent 只能按 instruction 字面执行。因此，Cron 任务的 prompt 必须按“无人值守作业书”来写，而不是聊天指令。

## 问题

交互式 prompt 依赖追问和确认，而 Cron 不支持这些。常见故障包括：

- 时间表达模糊：“早上”导致时区混乱。
- 输出目标不明：结果只留在对话里，没人看。
- 失败静默：异常不告警。
- 缺少幂等：重叠触发或重试造成重复执行。
- 权限过宽：写“有异常就修复”，可能触发危险操作。

例如，一个每小时检查服务状态的任务，instruction 写“检查服务是否正常，异常就处理”。实际运行时，Agent 可能只回复“服务看起来正常”，没有记录、没有检查具体端点，更没有告警。

## 做法/步骤

把 Cron 任务当成一次性脚本，固定字段。建议模板如下：

```text
[Objective] 一句话目标
[Schedule] 触发时间，使用 IANA 时区（如 Asia/Shanghai）
[Input] 读取的文件、URL、环境变量、历史状态
[Steps] 不超过 5 步，动词开头，避免歧义
[Output] 写入路径 + 格式 + 通知通道
[Failure] 失败时通知谁、是否重试、禁止扩大操作
[Idempotency] 开始前检查锁文件或上次结果
```

Before/After 对比：

Before：
```text
每天早上帮我看看服务器状态
```

After：
```text
[Objective] 检查生产服务器状态并输出日报。
[Schedule] 每天 08:00 Asia/Shanghai。
[Input] 读取 .env 中 SERVER_LIST，使用只读 SSH 账号。
[Steps]
1. 逐个 ping 并检查 HTTP 200，超时 5s。
2. 记录失败节点和耗时。
3. 不执行重启或修改操作。
[Output]
写入 ./reports/server-YYYY-MM-DD.md，并发送到 #ops 频道。
[Failure]
SSH 失败立即告警到 #ops，停止任务。
[Idempotency]
若 ./reports/server-YYYY-MM-DD.md 已存在则跳过。
```

上线前先手动触发一次，观察输出是否符合预期；再开启 Cron。

## 踩坑点

- **时区**：Cron 系统常用 UTC，instruction 写“早上”会错位。统一写 IANA 时区，不用模糊词。
- **未检查外部状态**：抓取类任务需检查是否已抓过，否则重复。
- **输出丢失**：定时执行时没人看聊天窗口，结果必须落盘或推送到固定通道。
- **权限过大**：默认只读/只报告，写操作列白名单。
- **重复执行**：用锁文件或检查上次结果文件。
- **失败静默**：失败路径必须显式通知。
- **上下文污染**：有记忆的 Agent 可能被历史干扰，instruction 加一句“忽略之前对话，仅依据本指令”。

## 可复用建议

1. **固化模板**：每个 Cron 任务都包含上述字段，不要直接复制对话 prompt。
2. **时区与时间**：指令中显式写 IANA 时区；日志存储 UTC，展示本地。
3. **输出统一**：优先 JSON 或 Markdown 文件，便于审计。
4. **先 dry run**：加“只输出执行计划，不实际操作”，验证后再切换。
5. **超时与重试**：为任务设置超时和最大重试次数，避免卡死。
6. **敏感信息**：用环境变量注入密钥，不写进 instruction。
7. **运行日志**：记录每次开始、结束、结果摘要，方便排障。

## 总结

Cron 任务本质是无人值守的确定性作业，instruction 需要从“对话”转向“作业书”。好的 Cron prompt 是自包含、幂等、可观察、失败会叫的。少写花哨 prompt，多写明确约束，稳定性和可维护性会明显提升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3e57a26b99c090e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/58f67a341296f621.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/ae9545d004920505.png)

