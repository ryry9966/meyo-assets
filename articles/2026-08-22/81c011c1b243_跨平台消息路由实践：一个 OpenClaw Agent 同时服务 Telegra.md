---
title: 跨平台消息路由实践：一个 OpenClaw Agent 同时服务 Telegram 与 Discord
feedId: 34153
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

最近在给一个 OpenClaw Agent 加渠道时，遇到一个典型问题：Agent 已经在 Telegram 上稳定运行，但团队一部分讨论发生在 Discord，希望在两边都能使用同一个 Agent，而不是复制两套逻辑。

如果只是“再写一个 Discord bot”，并不难。难点在于：同一个 Agent 运行时、同一套 tools、会话状态和权限控制，要同时面对两个事件模型差异很大的平台。本文记录一次实际接入过程，重点放在消息抽象、路由和踩坑上。

## 问题

跨平台接入并不是简单的“多一个入口”。主要差异包括：

- Telegram 有 message、callback_query、edit_message 等事件；Discord 以 message_create、interaction、thread 为主。
- Telegram 群聊通过 `chat_id` 区分，Discord 则需要处理 guild、channel、thread、forum。
- 两边消息长度、附件、回复语义、编辑和删除通知都不一致。
- 速率限制和重试策略不同：Telegram 单聊天约 1 条/秒，Discord 有全局和路由级限流。
- 用户身份无法自动对应，同一个人的 Telegram ID 和 Discord ID 没有关联。

如果 Agent 核心逻辑直接依赖平台字段，后续维护成本会很高。

## 做法

### 1. 先定义统一消息 Envelope

平台适配层只做转换，Agent 核心只认识一种结构：

```json
{
  "event_id": "tg_123456",
  "platform": "telegram",
  "chat_id": "-100xxxx",
  "thread_id": null,
  "sender_id": "user_abc",
  "text": "帮我查一下构建状态",
  "attachments": [],
  "reply_to": null,
  "timestamp": "2025-05-01T10:00:00Z"
}
```

`event_id` 用于幂等去重；`thread_id` 在 Telegram 里通常为空，在 Discord 里来自 thread 或 forum post。这样上层就不需要关心平台差异。

### 2. 接入层做平台适配

Telegram 侧使用 Bot API long polling 或 webhook，把 `message`、`edited_message`、`callback_query` 转成 Envelope。Discord 侧使用 gateway 接收 `message_create` 和 interaction，再统一转换。

适配器要保持薄，只做字段映射和事件过滤。比如 Discord 的 bot mention 可以转成直接触发，Telegram 群聊则默认只响应 `/` 命令或 mention，避免误触发。

### 3. 路由与会话管理

路由层根据 `platform + chat_id + thread_id` 选择一个会话。关键是不要让 Discord 的同一个 channel 下不同 thread 互相污染上下文。OpenClaw 的会话状态最好外置到 Redis 或 SQLite，不要放在进程内存中。

对于需要同时服务多个群组的 Agent，建议维护一个 `active_chats` 配置，只处理允许的 chat_id 或 guild_id。

### 4. 发送层与错误隔离

回复时通过 outbound adapter 返回。Telegram 超过 4096 字符需要分段；Discord 普通消息限制是 2000 字符，也需要分片。发送失败不能直接打断 Agent 主流程，应该进入重试队列，只记录错误。

## 踩坑点

**编辑事件很容易被忽略。** Telegram 用户修改消息会触发 `edited_message`，Discord 也会推送 update。如果 Agent 不处理编辑，不会有大问题；但如果要做“重新回答”，必须显式决定哪些平台支持编辑重跑。建议第一版直接忽略编辑事件，只处理新消息。

**Discord thread 场景容易串上下文。** 同一个 channel 下多个 thread 的消息，如果只看 `channel_id`，不同对话会混在一起。必须把 `thread_id` 纳入会话 key。Telegram 的 forum topic 也有类似的 `message_thread_id`，需要同样处理。

**限流不是只加 sleep。** Discord 的 429 响应会带 `retry_after`，需要尊重全局和路由级限流。Telegram 方面，连续发送到同一个 chat 需要本地队列，否则容易出现 429。建议每个目标 chat 维护一个最小间隔队列，而不是用 `await asyncio.sleep(1)`。

**事件重复投递。** Webhook 在某些情况下会重试，导致重复消息。如果同一条消息触发两次 Agent，可能造成重复工具调用。因此 `event_id` 去重非常必要，可以用 Redis SET 或数据库唯一约束。

**身份映射不能自动完成。** 同一个用户在 Telegram 和 Discord 上可能使用不同昵称和 ID。如果 Agent 需要做权限判断，建议在配置里显式绑定 `telegram_user_id` 和 `discord_user_id`，或者让用户在任一平台执行一个绑定命令。

## 可复用建议

- 平台适配层不要放业务逻辑，只做转换、过滤和限流。
- 所有进入 Agent 的消息都使用统一 Envelope，并把 `event_id` 作为幂等键。
- 会话状态外置，避免平台切换时丢失上下文。
- 第一版只支持文本和图片附件，暂不处理 sticker、reaction、语音转写等平台特性。
- 监控三个指标：入站消息数、发送失败数、队列等待时间。这比只看 error log 更早发现问题。

## 总结

同时服务 Telegram 和 Discord 的核心不是“多写两个 bot”，而是把平台差异隔离在薄适配层内。Agent 只关心统一消息结构、会话状态和工具调用结果。这样后续接入 Slack、Matrix 或 WebChat 时，也只需要增加一个 adapter，而不需要改动 Agent 核心逻辑。

这种模式在 OpenClaw 生态里比较适合自动化类 Agent：告警查询、部署状态、知识库问答等。只要消息抽象做好了，平台扩展成本会显著降低。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/108b82caa2fbd9e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3b46332d0386c306.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/10dc81af05be0549.png)

