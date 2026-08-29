---
title: Prompt 工程实战：Cron 任务 instruction 怎么写才不跑偏
feedId: 35196
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 agent 自动化环境里，Cron 任务并不是传统意义上的“定时执行脚本”，而是定时把一段 instruction 交给 agent 当初始任务跑。比如每天定时“看一下某个 API 状态”“整理收件箱”“生成日报”。

问题在于：Cron 执行时没有人在旁边确认，也没有交互纠偏。instruction 质量直接决定任务是稳定运行，还是每天制造一个需要人工收拾的新问题。

## 问题

很多人把 Cron 指令写成一句话：

> 每天早上帮我检查一下服务器状态，有问题就处理一下。

这种写法在三方面不可控：

- 目标模糊：什么叫“检查”，什么叫“问题”，什么叫“处理”；
- 动作越界：Agent 可能自行重启服务、改配置、删文件；
- 输出不可用：结果可能是一大段自然语言，没有固定格式，也没有失败通知。

本质上，Cron 的 instruction 不是聊天，而是“定时运行手册”。它要接近一个小型确定性程序。

## 做法：把 instruction 写成运行手册

我一般按七个字段写：

1. **定位**：一句话说明本次任务身份，例如“你是每日 09:00 的数据库备份检查任务，只执行一次”。
2. **目标**：单一、可验证。例如“仅检查备份目录中最近 24 小时是否产生新文件，不要修复、不要清理”。
3. **输入与数据源**：明确路径、时间范围、环境变量、API 地址。不要写“今天”，写“运行时刻前 24 小时”。
4. **执行步骤**：编号列出，禁止跳过或自行扩展。
5. **输出与副作用**：指定输出目的地、格式；写操作必须显式授权，默认只读或预演。
6. **边界**：不允许做清单之外的动作，不要搜索网络、不要调用额外工具，除非明确列出。
7. **完成与失败标准**：明确 DONE/FAILED，失败时记录日志并通知。

一个可参考的 instruction 模板：

```text
Role: daily backup check, run once.
Goal: verify that at least one backup file was created in /data/backup in the last 24h.
Inputs: only local directory /data/backup, current time from system.
Steps:
1. Run `ls -lt /data/backup | head -n 5`.
2. Compare the newest file mtime with current time.
3. If age <= 24h, output DONE: newest_file=<name> age_minutes=<n>.
4. If no file or age > 24h, output FAILED: reason=<...>.
Output: append one line to /var/log/backup_check.log, and send a notification only on FAILED.
Boundary: do not delete, move, or repair any file. Do not use network search.
Failure policy: on command error, record FAILED with stderr truncated to 200 chars, then exit.
```

这个例子没有硬编码密钥，也不依赖上一轮记忆。

## 踩坑点

- **“检查并修复”太危险**：定时任务无人值守，写操作必须单独授权。先跑只读模式，稳定后再逐步放开。
- **时间表达模糊**：用“今天”“最近”会导致不同运行时刻结果漂移。改成“运行时刻前 24 小时”或直接传日期参数。
- **没有指定输出位置**：Agent 可能会把结果写在聊天流里，但 Cron 没有接收者，等于白跑。指定日志文件或通知渠道。
- **上下文过期**：Cron 任务每次应自包含，不要依赖“上次说过”“之前的文件还在不在”。需要的信息写在 instruction 里，或从数据源重新读取。
- **失败静默**：必须设计失败输出。最好最后一行固定是 `DONE` 或 `FAILED`，上层才能做监控。
- **密钥写进 instruction**：指令可能被记录、同步、截图。密钥放环境变量或 secret 管理里。
- **并发与重复触发**：任务执行时间超过 Cron 间隔会重叠。需要幂等设计，比如输出文件加锁、写日志用 append、检查已有实例。

## 可复用建议

- 把 Cron instruction 纳入版本管理，变更留痕；
- 上定时前先手动跑 3 次，看输出是否稳定；
- 如果模型支持，固定低温度参数，减少随机性；
- 每次运行输出一行 `key=value` 或 JSON，方便后续解析；
- 对关键任务做“先预演后执行”：第一步生成动作清单，第二步再执行；
- 让 Agent 只做“判断 + 通知”，把有副作用的动作交给独立脚本或插件。

## 总结

Cron 任务的 instruction 不应该是一段模糊的自然语言，而应该是“给 Agent 的定时运行手册”：目标单一、上下文自包含、步骤可验证、边界清晰、失败可见。

写好后，你得到的不是一个更聪明的 Agent，而是一个更可预测的自动化单元。这也是 Prompt 工程在自动化场景里的核心价值：不是让模型说得更好，而是让它每次都做得一样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/912e8eb78e384975.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/63ea937ec6823c45.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c181da896a1ac368.png)

