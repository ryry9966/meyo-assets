---
title: 给“失忆”的 Agent 写提示词：Cron 任务 instruction 实战
feedId: 36605
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

OpenClaw 的 cron 功能经常被当成“定时提醒”用：挂个 schedule，写一句“每天早上八点发日报”，完事。跑几天就会发现结果时好时坏：有时是表格，有时是流水账；有时数据看着不对劲；有时干脆静默没输出。

问题通常不在 schedule，在 instruction。

## 问题：触发时的 Agent 是“失忆”的

关键认知：cron 触发的那次运行，**不继承你当前对话的上下文**。你写 instruction 时脑子里默认的那堆背景——“我的仓库”“常用的那个 MCP”“上次说的格式”——执行时统统不存在，Agent 拿到的只有 instruction 本身和系统配置。

所以 cron instruction 的本质是：给一个能力很强但完全失忆的同事，留一张足以独立干活的交接便条。用这个标准衡量，大多数一句话 instruction 都不合格。

## 做法：六段式模板

我现在的 instruction 基本固定成六段，可直接套用：

1. **任务一句话**。目标说清，不铺垫。
2. **数据来源**。明确用哪个工具/MCP/命令，参数是什么。这是最常被省略的一段，也是输出质量方差最大的来源。
3. **步骤**。逐条列出，多步骤任务尤其需要。
4. **输出格式**。给模板，哪怕只是“三行以内：结论/数据/异常”。格式固定，多天的结果才可对比。
5. **异常分支**。拿不到数据怎么办？部分失败怎么办？明确写“如果 X 失败，说明原因，不要编造”。
6. **交付方式**。发到哪个渠道/session，要不要标记紧急。

写完先手动触发一次再挂 schedule：支持 run-now 就直接跑，不支持就把同一段 instruction 贴进对话执行。观察两三天的真实输出，回来改 instruction，一般三轮能稳定。

## 踩坑点

- **时区**。服务器跑在 UTC，你写“早上八点”，实际触发可能是下午四点。schedule 的时间语义要和时区一起确认。
- **相对时间**。instruction 里别写“今天早上”——它相对的是执行时刻，天然有歧义。时间交给 schedule 定义，instruction 只管“做什么”。
- **让 Agent 自己找数据**。不指定工具，每次运行路径都可能不同，而且失败是静默的。要指定来源，并给兜底方案。
- **静默编造**。最危险的一种失败：任务“正常”结束，数据却是模型编的。异常分支那段不是可选项。
- **一个 job 塞多件事**。日报＋提醒＋备份塞一起，instruction 越长越顾此失彼。先拆成多个小 job，各自稳定后再考虑组合。

## 可复用建议

- 把每条 cron instruction 当独立 prompt 审核：假设读者对你们的对话一无所知。
- 输出格式定了就别轻易改，改动会破坏跨天对比。
- instruction 会腐烂：工具换了、渠道变了，它不会自己更新。建议每月扫一遍所有 job。
- 复杂任务先在对话里调通 prompt，再固化进 cron，顺序不要反。

## 总结

Cron 任务的质量上限，基本由 instruction 决定。schedule 负责“什么时候”，instruction 必须独立定义“做什么、用什么做、做成什么样、失败了怎么办”。把它当成给失忆同事的交接文档来写，用真实输出持续迭代，大部分“时好时坏”的问题会自然消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/f617c4dd54e33d0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/4ccc3031ea9392ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/36394822e024faa7.png)

