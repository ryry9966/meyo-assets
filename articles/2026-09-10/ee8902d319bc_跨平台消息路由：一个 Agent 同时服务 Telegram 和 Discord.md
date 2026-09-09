---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36802
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

我们组的情况很典型：一半同事长期泡 Telegram，另一半在 Discord 上维护社区。Agent 最初只接了 Telegram，Discord 侧的人要么切平台、要么干脆不用。需求很直接：**一个 Agent 实例、一份配置和记忆，同时服务两个渠道**——而不是起两套各管各的。

## 问题：不是"再加一个 channel"

接入层本身不难，OpenClaw 的 channel 插件把轮询和鉴权都封装好了，真正的工作量在路由语义上，拆开是三件事：

1. **身份与会话**：同一个 Telegram user 和 Discord user 怎么被识别为同一个人？两个群的上下文要不要隔离？
2. **格式适配**：Telegram 的 MarkdownV2 和 Discord 的 markdown 方言互不兼容，长度限制也不同（4096 vs 2000）。
3. **运维细节**：媒体文件、限流、重试幂等，每个平台各有一套脾气。

## 做法

**1. 单实例双渠道。** Gateway 只跑一个进程，注册 telegram 和 discord 两个 channel，共享同一个 Agent 核心和 MCP 工具层。切忌起两个实例：同一个 bot token 两处 polling 会互踢（getUpdates 409），Discord 侧则表现为重复回复。

**2. 会话隔离 + 记忆共享。** 默认每个 chat 独立 session，避免跨群串台；跨平台的"共同记忆"走 Agent 长期记忆和 workspace 文件，而不是硬共享 session。同时在 system prompt 注入渠道标识（"当前消息来自 discord #dev"），让 Agent 自动调整 @ 提及和语气习惯。

**3. 身份映射表。** 维护 Telegram user id ↔ Discord user id ↔ 内部主身份的绑定关系。鉴权白名单、配额、通知订阅全部挂主身份，同一个人不会被当成两个用户限权。

**4. 出站统一、出口适配。** Agent 产出标准 Markdown，由各 channel 的 formatter 负责转换和分片。分片时感知 fenced code block 边界，宁可一条短一点，也不要把代码块切成两半。

**5. 灰度。** 先只接一个测试群和测试频道跑一周，确认日志里没有串台和重复回复，再放量。

## 踩坑点

- Telegram MarkdownV2 转义是重灾区，`_ * [ ]` 在用户名和代码里高频出现，一处漏转义整条消息 400。后来统一切到 HTML parse mode 才稳定。
- Discord 附件 URL 带时效签名，直接存 URL 过几天就失效。正确做法是收到后落本地，Agent 引用本地路径。
- Webhook 重试会导致 Agent 对同一消息回复两次，按平台消息 id 做去重窗口（我们设了 5 分钟）。
- Discord 侧批量推送极易触发 429，出站队列加指数退避，别裸发。

## 可复用建议

- **渠道适配层保持薄**：只做鉴权、格式、分片、去重；业务逻辑全放 Agent 核心。以后接第三个渠道就是复制一层薄壳。
- **平台差异配置化**：消息长度上限、parse mode、mention 语法写成 per-channel 配置，不要散落在代码里。
- **日志统一带 channel / session / user 三个字段**：双渠道排障八成靠这个。
- **先想清楚要不要共享记忆，再想怎么共享**。多数场景"会话隔离 + 记忆共享"就是答案，别一开始就做全局路由。

## 总结

这件事的难点从来不在"接入第二个平台"，而在身份、会话、格式三个路由语义问题。设计期把这三件事想清楚，双渠道就退化成一个配置问题；想不清楚，每接一个渠道都是一次重写。我们的结论是三条：**核心厚、适配薄、身份统一**——守住了，跨平台只是顺手的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/7e12c7ea308699e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c301450ea03f685d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/9334a7c6dabc1c30.png)

