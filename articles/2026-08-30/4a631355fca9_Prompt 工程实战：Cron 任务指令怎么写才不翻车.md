---
title: Prompt 工程实战：Cron 任务指令怎么写才不翻车
feedId: 35288
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 里挂 cron 任务，很容易把 instruction 写成一次性对话：要么只写一句“每天检查 X”，要么把整页业务背景塞进去。定时任务执行时没有用户追问，模型只能按文本做决定，写错代价是重复执行、误写数据、静默失败。

## 问题

Cron instruction 与普通 prompt 有三种本质差异：

1. 无人澄清：触发时不在交互上下文里。
2. 边界模糊：任务允许读什么、写什么、发什么，必须提前约束。
3. 状态敏感：多次执行之间可能有状态，不能用对话记忆承载。

所以 cron 任务指令要更像“运维 runbook”，而不是“聊天需求”。

## 做法：按 runbook 结构写 instruction

建议至少拆成 6 段：

- Trigger：写清 cron 表达式和时区。
- Objective：一句话说明成功结果。
- Preconditions：执行前必须满足的条件。
- Steps：有限步骤，动作要可验证。
- Output/Write：明确产物位置和允许写入范围。
- Failure：失败时怎么处理、是否重试、通知到哪里。

示例：

```text
[Trigger]
cron: 0 9 * * *
timezone: Asia/Shanghai

[Objective]
拉取数据源 X 昨日新增记录，追加写入 data/x_daily.json，不发送任何消息。

[Preconditions]
- MCP tool fetch_source_x 可用
- 文件 ./data/x_daily.json 存在且可写

[Steps]
1. 读取 ./state/x_last_sync.json 中的 last_sync_at。
2. 调用 fetch_source_x(since=last_sync_at)。
3. 若返回为空，更新 last_sync_at 后结束。
4. 对每条记录做 schema 校验，丢弃无 id 或 timestamp 的条目。
5. 将有效记录追加写入 data/x_daily.json。
6. 更新 last_sync_at 为本次最大 timestamp。

[Output]
- 仅允许写入 data/x_daily.json 与 state/x_last_sync.json。
- 不发送消息、不修改其他文件。

[Failure]
- 若工具调用失败，输出错误原因并停止。
- 不自动重试超过 1 次。
```

这个结构看起来繁琐，但它减少了模型在无人确认时的“自由发挥”。

## 踩坑点

1. **时区不写，执行时间漂移**：很多系统默认 UTC，国内业务写成 `0 9 * * *` 但不声明时区，可能在北京时间下午才跑。Trigger 里必须单独写 timezone。

2. **状态放在对话记忆里**：模型重启或新会话后，`上次我已经处理到第 5 条`这类记忆丢失。状态必须外置到文件、数据库或 MCP 资源，指令里只保留读取/更新状态的动作。

3. **“定期检查”太笼统**：如果步骤是“检查服务是否正常”，模型可能只调用一次工具就结束，甚至只看历史结果。要写成“调用 health_check，若 status != ok，则执行 X”。

4. **写操作没有边界**：自动任务里不要写“整理文件”“清理无效数据”这种开放目标。必须限定目录、文件类型、最大改动数，否则一次误判可能删掉大量数据。

5. **失败静默**：没有 Failure 段的 cron 任务，失败后往往没人知道。至少要写“失败时输出错误并停止”，如果平台支持通知，再指定通知目标。

6. **cron 表达式和描述不一致**：例如描述写“每个工作日”，表达式却写成 `0 9 * * 1-5`，实际没错，但容易和“每天”混。改任务时同步改 description。

## 可复用建议

- 先用只读步骤验证，再允许写入。可以让 instruction 先执行 plan，输出将要写入多少条、写到哪个文件，再继续；或强制 dry-run。
- 给任务加版本号和变更记录。cron 指令一旦稳定，不要频繁微调。每次修改要手动触发一次看结果。
- 状态文件尽量小，只存必要游标，如 `last_sync_at`、`last_id`。
- 使用 MCP 工具或插件工具时，把工具名写死，不要只写“用合适的工具”。工具选择错误是最常见的失败源。
- 模板化：把你的 cron 任务模板沉淀到团队仓库，比每次重写更可靠。

## 总结

Cron 任务的 instruction 不是写得更“聪明”，而是写得更“确定”。好的定时指令应让代理在无用户环境下，仍能回答四个问题：什么时候跑、做什么、写到哪、失败怎么办。把 instruction 当成可审查的 runbook，而不是一段期望，能显著降低自动任务翻车率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b35271d30005b1bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/96803e4239d3fdfc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2ad3f9a1b8ff6896.png)

