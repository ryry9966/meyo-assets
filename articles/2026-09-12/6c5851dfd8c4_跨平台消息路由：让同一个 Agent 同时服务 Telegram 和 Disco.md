---
title: 跨平台消息路由：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 37231
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

Agent 从玩具走向可用，第一道坎往往是入口问题。团队一半人泡在 Telegram 群，另一半社区在 Discord，让用户迁就 Agent 不现实，只能让 Agent 同时出现在两边。这篇帖子记录我们接入双平台时路由层的设计决策与踩坑，代码不多，坑不少。

## 问题本质

同时接两个平台，难点不是"调两套 API"，而是两边消息模型的语义差异：

- **身份体系不同**：同一用户在两个平台是两个 identity，权限和记忆是否打通需要明确决策；
- **消息格式不同**：Telegram 用 HTML/MarkdownV2，Discord 是自己的 markdown 方言，表格、引用、@提及互不兼容；
- **会话语义不同**：Discord 有频道和 thread，Telegram 是 chat + reply，会话切分粒度天然不一致。

不做统一抽象的话，每个新功能都得在两边各写一遍适配逻辑，维护成本随平台数线性增长。

## 做法

采用经典的"规范消息模型 + 薄适配器"结构，分四步：

1. **定义 Canonical Message**。内部只认一种消息结构：发送者、会话键、文本、媒体、reply 引用、原始 payload。所有平台消息入站先归一化到这个模型。
2. **入站适配器只做翻译**。Telegram polling worker 和 Discord gateway worker 各自把原始事件转成 Canonical Message，推入同一条内部队列。之后 Agent 完全不知道消息来自哪个平台。
3. **出站渲染按平台走**。回复统一进出站队列，由各平台 renderer 转成对应格式；渲染失败降级为纯文本重发，保证消息不丢。
4. **会话键与路由规则**。会话键设计为 `platform:chat_id`，一个平台会话对应一个 agent session，第一版刻意不做跨平台合并。回复永远"原路返回"：哪条入站消息触发的，就从哪个平台回。

部署形态是单进程两个 gateway worker + 一个 agent worker，队列用 Redis 顶住，量大了再拆。

## 踩坑点

- **Markdown 方言是重灾区**。代码块、链接、@提及两边语法都不同，最终为 renderer 维护了一张平台能力矩阵，超出能力的语法直接降级。
- **Telegram 解析失败很静默**。MarkdownV2 转义漏一个字符，消息可能直接发不出去且报错不直观。解法是渲染后先本地校验，再带纯文本 fallback。
- **限流收口在出站队列**。两边限流策略差异大（Discord 的 429 带 retry_after，Telegram 是 429 + 参数），统一在出站队列做退避，业务层不感知。
- **Discord 交互 token 只有 15 分钟**。长任务别指望编辑原消息汇报进度，改成发新消息或完成后补发。
- **过滤回环**。Agent 自己发的消息、其他 bot 的消息，都要在入站适配器丢弃，否则容易自我对话刷屏。
- **媒体差异**。文件大小上限、语音格式、贴纸支持都不同，媒体统一走对象存储中转，平台侧只收链接或重传压缩版。

## 可复用建议

- 适配器保持"薄"：只做格式翻译，不做业务判断，业务逻辑全在 Agent 侧；
- 用录制的真实消息做契约测试 fixture，平台 API 变更时跑一遍就知道哪里坏了；
- 出站消息带幂等键，重试不会重复发送；
- 上线前加 dry-run 模式，只打印路由决策不真发，排查路由问题效率翻倍。

## 总结

跨平台路由的核心是把"平台差异"压缩在适配层，让 Agent 面对唯一的消息模型。第一版我们刻意不做跨平台身份合并、不做复杂扇出，先让链路稳定可观测，再逐步演进。一句话总结：**路由层越无聊越好，差异处理得越早越省**。欢迎在社区贴出你们的平台能力矩阵，一起补全这份对照表。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/0b41be7ef18d825c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/c456f87a7a22a2f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/61b4ee6b62690151.png)

