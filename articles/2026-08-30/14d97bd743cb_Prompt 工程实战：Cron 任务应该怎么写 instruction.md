---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 35379
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 或类似 Agent 自动化环境里，Cron 任务通常不是“到点提醒我”，而是“到点替我把某件事干完”。比如定时备份、定时拉取数据、定时巡检服务状态。  
但实际操作中，很多人把 Cron 任务的 instruction 写成了聊天指令：“每天晚上帮我备份一下项目目录”。结果任务跑起来，要么重复执行，要么静默失败，要么执行了但没人知道它到底做了什么。

问题的核心在于：Cron 任务是一次性的“无人值守执行”，而不是一场有来有回的对话。指令写得越随意，执行结果越不可控。

## 问题

Cron 任务的 instruction 和普通对话 prompt 有几个本质区别：

1. **冷启动上下文**：每次触发都是一个新会话，Agent 不会自动记住上次执行到哪了。
2. **无人监督**：执行时你大概率不在旁边，失败不会被立刻发现。
3. **需要幂等**：定时器可能因为重试、手动触发、系统补跑等原因重复执行，指令必须能避免脏写或重复副作用。
4. **可观测性差**：如果只输出一句“完成了”，你事后无法判断它到底做了哪些动作、花了多久、有没有部分失败。

所以，Cron 任务的 instruction 需要像写运维脚本一样：前置检查、执行动作、输出契约、异常处理、幂等保护，一个都不能少。

## 做法/步骤

下面是一个可复用的 Cron instruction 模板。以“每日凌晨 2 点备份项目目录到远端存储”为例。

```text
[Cron Task Instruction]
schedule: "0 2 * * *" (UTC+8)
objective: 备份 /home/user/projects 到远端存储，并校验文件数
context:
  - 工作目录: /home/user/projects
  - 依赖: rclone 已配置，远端名为 backup-remote
preconditions:
  - 检查 rclone 是否可用
  - 检查上次备份是否成功
steps:
  1. 读取 /home/user/.cache/backup_state.json 获取上次备份时间戳
  2. 执行 rclone sync /home/user/projects backup-remote:projects-backup
  3. 校验远端文件数是否大于 0，且与本地文件数偏差不超过 5%
  4. 更新 backup_state.json 中的 last_success_ts 和 last_status
output_contract:
  - 输出 JSON: {"status": "success|skipped|failed", "files_synced": 0, "duration_sec": 0, "error_code": ""}
failure_handling:
  - 若 rclone 不存在，输出 {"status":"failed","error_code":"DEPENDENCY_MISSING"} 并终止
  - 若远端不可达，重试 2 次，每次间隔 30s
  - 若校验失败，输出 {"status":"failed","error_code":"VALIDATION_FAILED"}
idempotency:
  - 若同一天内已存在 last_success_ts 且日期相同，则跳过执行，输出 {"status":"skipped"}
```

这个模板之所以有效，是因为它把“怎么做”“怎么算成功”“失败了怎么办”都写死在指令里。Agent 不需要发挥创意，只需要按部就班执行。

## 踩坑点

1. **时间表达模糊**  
   写“每天晚上”或“每隔一阵子”，Agent 可能按它自己的时区理解，或者干脆解析成奇怪的时间。直接用 cron 表达式并标明时区，是唯一可靠的做法。

2. **只描述动作，不定义输出**  
   很多人只写“帮我备份”，不写“输出什么”。结果任务跑完了，日志里只有一句“备份完成”。你连它备份到哪个路径、花了多久都不知道。一定要定义 output_contract，最好是 JSON 或固定 schema。

3. **没有幂等保护**  
   Cron 任务可能因为调度器重试、手动补跑、系统重启等原因在同一天内多次触发。如果不写“检查上次执行状态”，它可能重复备份、重复发消息、重复写文件。用状态文件或数据库记录 last_run_ts，是成本最低的幂等方案。

4. **把失败处理丢给 Agent 自己判断**  
   “如果失败了你就想办法重试”这种话对 Agent 来说太宽泛。它可能无限重试，也可能干脆放弃。明确写出“重试 2 次，间隔 30s，之后输出 ERROR_CODE”才能保证行为可预期。

5. **忽略冷启动上下文**  
   Cron 触发的会话不会自动加载你之前的对话记忆。如果你需要依赖“上次执行结果”，必须在 instruction 里显式告诉它去读哪个文件、查哪个表。否则它每次都是“从零开始”。

## 可复用建议

- **把 Cron instruction 当成“SOP + 运维脚本注释”来写**，不要当成聊天。它应该包含 precondition、action、output、rollback/failure。
- **固定输出 schema**。每次执行都输出统一格式的 JSON，方便后续接入日志分析或告警。
- **用状态文件做幂等**。哪怕只是一个带时间戳的 JSON 文件，也能避免大部分重复执行问题。
- **在 instruction 里写明“请先检查上次执行结果”**。这比依赖 Agent 的记忆可靠得多。
- **每次执行后留下证据**。例如把输出 append 到日志文件，或写入固定路径的 result.txt。方便你事后排查“为什么这次没跑成功”。
- **保持 instruction 简洁但不省略关键约束**。不要写冗长的背景故事，但也不能只写一句“帮我备份”。把“目标—步骤—输出—异常”四段写清楚即可。

## 总结

Cron 任务的 instruction 核心不是“告诉 Agent 做什么”，而是“告诉 Agent 如何在无人监督下可靠地完成，并且留下可审计的证据”。  
好的 Cron instruction 具备三个特征：可重放、可审计、可恢复。  
下次你写定时任务时，可以试着把它想象成在给一个刚入职、没有上下文、但执行能力很强的运维同事写交接文档——这样写出来的 instruction，通常不会跑偏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/8f96c8697f6a4a5f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d2ecb4faa451bee5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b925fe9d7cbbc1fc.png)

