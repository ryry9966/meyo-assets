---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34401
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

如果你用 OpenClaw 跑个人 Agent，常见需求是：把同一个助手接到 Telegram 和 Discord，不想为每个平台各维护一套人设、工具权限和上下文。单渠道接入通常很快，但真正落到日常使用，问题往往在“路由”而不是“接通”。

## 问题

一个 Agent 同时服务两个平台，核心挑战不是回调配置，而是：

- 消息从不同平台进来，结构不同，不能直接把原始 payload 交给 Agent。
- 同一个用户在两个平台的 ID 不同，身份如何统一。
- 上下文怎么隔离，避免群聊、DM、Discord 论坛串线。
- 平台格式差异：Telegram 的 Markdown/HTML、Discord 的消息长度限制和 embed。
- 限流、重试、消息 ID 映射，错了不容易排查。

## 做法 / 步骤

### 1. 接入两个 channel

在 OpenClaw 中分别启用 Telegram 和 Discord 网关，凭据放环境变量。不要为了省事把两个平台逻辑写进同一个大插件。

```yaml
channels:
  telegram:
    enabled: true
    bot_token: ${TELEGRAM_BOT_TOKEN}
  discord:
    enabled: true
    bot_token: ${DISCORD_BOT_TOKEN}
```

### 2. 定义统一的入站 Envelope

所有消息进入 Agent 前，先 normalize 成统一结构。这是整个路由里最值得做的一步。

```ts
type Envelope = {
  channel: 'telegram' | 'discord'
  chatId: string
  threadId?: string
  senderId: string
  messageId: string
  text: string
  mediaRefs: string[]
}
```

Telegram 的 `chat.id`、Discord 的 `channel_id` 都映射到 `chatId`；Discord 的 thread id 或 Telegram 的 reply 关系，映射到 `threadId`。

### 3. 用 session key 隔离上下文

不要让 Agent 只按用户 ID 记忆，否则同一个用户在两个平台的 DM 会混在一起，同一个群里的不同话题也会串。用“平台 + chat + thread”做粒度：

```ts
const sessionKey = `${env.channel}:${env.chatId}:${env.threadId ?? 'main'}`
```

如果平台没有 thread 概念，至少保留 `channel:chatId`；如果是 Discord forum，必须带上 thread id。

### 4. 出站适配

Agent 返回后，不要直接发送。先过一层 per-channel adapter：

- Telegram：根据配置选择 `MarkdownV2`、`HTML` 或纯文本，转义特殊字符。
- Discord：限制约 2000 字符；按 1950 左右安全线拆分，避免切断代码块、链接或列表。
- 统一先生成普通文本或 Markdown 子集，再由 adapter 做平台化。

### 5. 队列与重试

两个平台都有严格的限流。不要同步等待 HTTP 返回结果后继续处理下一条消息。出站放到队列里，统一处理 429、5xx 和网络超时。Discord 的 rate limit 通常返回 `retry_after`，需要按路由存储，不要疯狂重试。

## 踩坑点

- **身份映射缺失**：很多人的第一步是直接信平台的 `sender_id`，一旦需要跨平台识别同一个人，后面要补很多映射。建议最早建立 `platform_user` 表。
- **把平台差异带进 Agent**：如果你在提示词里写“如果是 Discord 就……”，很快会失控。平台差异应放在 adapter，不进入核心 Agent。
- **Markdown 转义不足**：Telegram 的 MarkdownV2 对 `.`、`-`、`!` 等字符敏感，直接用 Markdown 很容易发送失败。adapter 里做 sanitize 比在 prompt 里要求模型“不要用特殊字符”可靠。
- **长消息拆分坑**：Discord 拆分不是简单截断。要按段落、代码块边界拆，不然 Markdown 结构坏掉，用户体验很差。
- **消息 ID 丢失**：OpenClaw 内部分配的消息 ID 与平台真实 ID 不一一对应时，后续要编辑、删除、引用回复都会失败。建议维护一个 outbound_id 映射。

## 可复用建议

1. **Envelope 先行**：所有 input 先 normalize，后续加平台只新增 adapter。
2. **adapter 可测试**：把 Telegram adapter 和 Discord adapter 做成独立模块，用 fixture 模拟入站/出站。
3. **出站队列独立**：不要在每个 plugin 里各写一套 retry。
4. **先只开 DM，稳定后再开群组**：群组里 at bot、权限、消息去重更复杂。
5. **记录 channel+thread 维度指标**：消息量、失败率、重试次数、延迟，方便识别限流和掉线。

## 总结

跨平台消息路由看起来像一个“接入问题”，实际是一个“模型边界问题”。核心是：统一内部协议、隔离上下文、平台化出站、分布式队列。把这两层做好后，加更多渠道只会增加一个 adapter，而不是重写 Agent。

以上配置和代码是基于 OpenClaw 常见 gateway/plugin 模型抽象出来的，具体字段名以你使用的版本为准，但结构可以直接复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/692901e3fd6f3e45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/77a5e2aac9bafbbb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/bc50c1fe284795f5.png)

