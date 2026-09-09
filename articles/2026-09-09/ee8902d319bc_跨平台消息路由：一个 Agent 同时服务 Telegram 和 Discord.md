---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36763
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

我们的 Agent 最初只跑在 Telegram 上，个人助手场景很顺。后来团队协作迁到 Discord，问题来了：再起一个 Agent 意味着两份 prompt、两份记忆、两套工具配置，还会出现“两边答案不一致”的尴尬。OpenClaw 的 Gateway 架构其实天然适合这件事——Agent 是单例，渠道只是适配器。真正的工作量不在“再接一个 bot”，而在中间那层路由。

## 问题

双渠道不是把 token 填两遍就完事，实际要解决四个问题：

1. **会话与身份**：同一个人在 TG 和 DC 上是两个身份，默认该隔离（`platform:chat_id` 作为 session key），什么时候允许合并？
2. **格式差异**：Telegram MarkdownV2 的转义出了名的脆，Discord 有 2000 字符硬限制和自己的 markdown 方言。
3. **媒体不对称**：语音消息、图片直链、CDN 过期，两边能力对不齐。
4. **主动消息**：heartbeat、定时任务触发时，发到哪个渠道？谁决定？

## 做法

拓扑保持简单：一个 Gateway，两个 channel adapter，共享同一个 Agent runtime。分五步：

1. **先定义内部消息 schema**。入站消息在 adapter 里归一化成统一结构：`text / media_refs / reply_to / thread_id / sender / platform / chat_id`。渠道私有特性（Discord 的 embed、TG 的 caption）留在 adapter，不往上游漏。
2. **session key 用 `platform:chat_id[:thread_id]`**。群聊必须带 chat_id，否则两个群的上下文会串。需要跨平台身份合并时，用显式 allowlist 做 identity mapping，不要上模糊匹配。
3. **出站走 per-platform renderer**。Agent 输出标准 markdown，渲染层负责：TG 侧做白名单转义（只保留代码块和粗体，其余转纯文本），DC 侧按 2000 字符在句子边界分片。
4. **媒体先落地再引用**。两边把附件下载到共享存储，给 Agent 传本地路径，不要传平台 CDN 链接——过期和鉴权问题会留到半夜爆发。
5. **灰度上线**。第一周 Discord 只做只读镜像（能收、能归档、不回复），确认路由日志没毛病再放开出站。

主动消息我们定了条硬规则：每个定时任务必须声明 `target_channel`，没声明就路由到默认渠道并打 warning，绝不广播。

## 踩坑点

- **MarkdownV2 转义**：正则补丁修不完，最后干脆白名单渲染，丢一点格式换稳定。
- **polling 和 webhook 同时开**：消息重复消费了一整晚才发现。渠道配置里加互斥检查。
- **typing 指示**：Discord 的 typing 60 秒过期，长任务要续；TG 的 sendChatAction 只活 5 秒。统一在出站层做心跳续期。
- **限速差异**：DC 是全局 + per-channel 双层限速，只按全局限速会被单频道打爆。

## 可复用建议

- **归一化要早，渲染要晚**：渠道怪癖全部关在 adapter 里，Agent 和 prompt 感知不到平台存在。
- 每个渠道挂 feature flag，出问题可以单渠道降级到只读。
- 路由决策打结构化日志（`in_platform → session_key → out_platform`），排查“为什么发到那边去了”全靠它。
- 给路由层写单测：跨渠道同 session、超长分片、媒体缺失，都是能确定性复现的 case。

## 总结

一个 Agent 服务多渠道，本质是把“理解消息”和“投递消息”拆干净。OpenClaw 的 adapter 模式给了正确的骨架，剩下的是工程纪律：统一 schema、严格的 session key、显式的路由规则。做完之后，新增一个渠道（比如 Slack）的边际成本大约是一个 adapter 加一个 renderer，两天以内能收工。这笔账算下来，早期在路由层多花的功夫都是值的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/fd89ff4c0b8a1008.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/38e7a9225ac7dae3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/6a88a737b90ac960.png)

