---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36773
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 的典型用法是“一个 Agent 绑一个渠道”。用得久了自然会遇到：白天在 Discord 服务器里协作，晚上在 Telegram 私聊问事。两个 bot 各一套会话、记忆和 prompt，上下文割裂，配置维护也是双份。这篇记录我们把同一个 Agent 同时接入 Telegram 和 Discord 的过程，重点在路由层设计，不讨论模型选型。

## 问题

拆开看其实是四件事：

1. **接入层差异**：Telegram 走 long polling，Discord 走 gateway WebSocket，连接生命周期管理完全不同。
2. **消息模型差异**：格式方言、长度上限（4096 vs 2000 字符）、附件、回复引用各不相同。
3. **会话归属**：同一用户跨两个平台，共享记忆还是各开会话。
4. **出站节奏**：两边限流规则不同，Agent 批量输出时很容易撞限。

## 做法

核心原则一句话：**适配器只做传输，不做智能**。

1. **统一消息信封**。定义 `InboundMessage` / `OutboundMessage`，字段包括 `channel`、`chat_id`、`user_id`、`text`、`attachments`、`reply_to`、`correlation_id`，两端适配器都归一化到这个结构。
2. **收敛适配器接口**为 `start() / stop() / send() / normalize()`。Telegram 侧用 `getUpdates` 长轮询（单实例部署，避开 webhook 冲突），Discord 侧交给官方网关库处理重连和心跳。
3. **中央路由器**按 `chat_id` 维护会话，调用 Agent 核心（LLM + MCP 工具），响应按 `correlation_id` 回到原始渠道。工具异步执行也没关系——回包查的是信封字段，不是“当前会话”。
4. **身份映射单独建表**：`platform_id → unified_user`。短期记忆按会话隔离，长期记忆（偏好、事实）按 unified user 共享，避免上下文串台。
5. **出站统一进队列**，每个渠道挂一个令牌桶限流器，超长消息按段落边界分片，不做硬切字符。

## 踩坑点

- Telegram 解析模式直接用 HTML，别碰 MarkdownV2，转义规则能把人耗一整天。
- Discord 后台记得开 MESSAGE CONTENT intent，否则收到的内容是空的且不报错，极易误判为路由问题。
- 自己发的消息必须过滤，否则 bot 回复自己进死循环。
- Telegram 同时开 polling 和 webhook 会 409 冲突；多实例部署要改 webhook + 单写者。
- Discord CDN 附件链接有时效和 header 要求，拉取失败应走重试路径，而不是当成 Agent 错误抛给用户。
- 会话完全共享实测体验很差：A 平台的问题被 B 平台的讨论污染。最终选了“会话隔离 + 记忆分层共享”。

## 可复用建议

- `correlation_id` 从入站打到出站日志，排障时一条链路追到底。
- 适配器保持“笨”，格式转换、分片、限流全放核心侧，将来加 Slack 只需实现四个方法。
- 录制真实入站消息做回放测试，比手写单测更能暴露格式问题。
- 每个渠道一个 feature flag，语音转写这类单平台能力按渠道开关。

## 总结

跨平台不是“多接一个 SDK”，而是把传输、会话、身份、限流四层拆干净。适配器薄、核心厚，路由靠信封字段而不是隐式状态——这套结构落定之后，接任何新渠道都是增量工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/5e0f2a0fb1c60144.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/612756de72e3b46f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/26d5c023b04ba476.png)

