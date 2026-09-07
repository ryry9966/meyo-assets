---
title: 一个 Agent 同时服务 Telegram 和 Discord：OpenClaw 跨平台消息路由实践
feedId: 36401
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

最早我只跑了一个 Telegram bot 接 OpenClaw，用得很顺手。后来社区协作搬到了 Discord，问题来了：同一份上下文、同一套技能，我要么在 Discord 再起一个实例，要么让一个 Agent 同时接入两边。考虑到工作区文件、记忆和技能配置都只有一份，我选择了后者——单网关、双通道。跑了一个多月，把经验整理如下。

## 问题

跨平台接入不只是“多填一个 token"，真正要解决的是三件事：

1. **会话边界**：Telegram 上的对话和 Discord 上的对话，是同一个 session 还是分开？
2. **消息格式**：Discord 单条消息上限 2000 字符，Telegram 是 4096，两者的 Markdown 方言也不同。
3. **触发策略**：两个平台都有群聊，如果每条消息都触发 Agent，噪音和 token 消耗都会失控。

## 做法与步骤

**第一步：单网关接入两个通道。** 在 `openclaw.json` 的 `channels` 里同时配置 `telegram`（BotFather 的 token）和 `discord`（Bot token，记得开启 Message Content Intent）。只跑一个 gateway 进程，Agent、工作区、技能插件全量共享。

**第二步：保持默认会话隔离。** OpenClaw 默认按「通道 + 聊天 ID」分 session，我保留了这一点：两边对话独立，互不污染。跨平台的“共享”交给工作区文件——`AGENTS.md`、记忆文件、技能脚本是全局的，长期知识自然打通。

**第三步：收紧触发条件。** 私聊走 pairing/allowlist 白名单；群聊一律设置“仅 @提及触发”。这一步比想象中重要，后面踩坑会展开。

**第四步：在 AGENTS.md 里写格式约束。** 明确告诉 Agent：回复用保守 Markdown（避免表格、标题、Discord 特有语法），单条回复控制长度，长内容主动分段或落盘成文件再给路径。

**第五步：看日志验证路由。** 改完配置盯一会儿 gateway 日志，确认每个平台的消息命中的是预期 session key、触发策略生效，再放心长期跑。

## 踩坑点

- **群聊全量回复**：Discord 某个群一开始没设 mention 门控，Agent 见消息就答，一天烧掉大量 token。立刻改成仅提及触发。
- **长回复被截断**：Discord 2000 字符限制导致回复腰斩。解法是在系统提示里限制单条长度，超过则存文件。
- **同 token 双实例冲突**：我曾为了测试另起一个进程连同一个 Telegram bot，直接 polling 冲突报 409。一个 bot token 只能绑一个实例。
- **Intent 没开**：Discord 后台忘了开 Message Content Intent，所有消息内容为空，排查了半天配置，其实是平台侧开关问题。

## 可复用建议

- **通道当薄适配层**：所有业务逻辑、人格、技能都放在 Agent 工作区，通道只负责收发。以后加 Slack/微信，改动面很小。
- **会话隔离 + 记忆共享**是跨平台的黄金组合，不要强行合并 session。
- **白名单先行**：新通道上线先最小白名单灰度，稳定再放开。
- **身份映射写进记忆**：同一个人在两个平台 ID 不同，我在记忆文件里记了一条映射关系，Agent 就知道“这两个是同一个人”。

## 总结

单 Agent 双通道的核心不是把消息接进来，而是想清楚**什么该共享、什么该隔离**：会话隔离保证上下文干净，工作区共享保证知识和能力一致，触发策略和格式约束保证体验可控。跑通之后，这套结构天然支持继续扩展第三、第四个平台。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/a151dcf95b57876c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/4b7f966ab666eb99.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/8fb1680a1932f4d5.png)

