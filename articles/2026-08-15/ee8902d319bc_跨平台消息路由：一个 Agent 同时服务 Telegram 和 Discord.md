---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 33201
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 场景里，Agent 往往作为常驻服务运行。用户分散在 Telegram、Discord，如果为每个平台单独部署一个 Agent，会割裂上下文、工具调用和记忆。目标很明确：一个 Agent 核心同时接 Telegram 和 Discord，但不写两套业务逻辑。

## 问题

两个平台的 API、事件模型、消息长度、附件、编辑/删除、权限和限流都不一样。如果在 core 里到处写 `if (platform === 'telegram')`，Agent 很快会不可维护。跨平台消息路由的本质，是把平台差异关进 adapter，给 Agent 一个平台无关的 I/O 契约。

## 做法 / 步骤

### 1. 定义统一入站 envelope

不要让平台消息直接进入 Agent。先统一成内部结构：

```ts
interface AgentEnvelope {
  platform: 'telegram' | 'discord';
  channelId: string;
  userId?: string;
  messageId?: string;
  text: string;
  attachments: Attachment[];
  edited: boolean;
  raw?: unknown;
}
```

### 2. Adapter 只做协议转换

Telegram adapter 可以用 grammY，Discord 用 discord.js。它们只负责把平台事件转成 `AgentEnvelope`，再把 Agent 输出转回平台消息。

```ts
// Telegram adapter 示例
bot.on('message:text', async (ctx) => {
  const env: AgentEnvelope = {
    platform: 'telegram',
    channelId: String(ctx.chat.id),
    userId: String(ctx.from?.id),
    messageId: String(ctx.message.message_id),
    text: ctx.message.text,
    attachments: [],
    edited: false,
    raw: ctx.message,
  };
  await inbound(env);
});
```

Discord 侧监听 `messageCreate`，但要注意 `content` 可能为空。

### 3. 路由层处理权限与分组

在 Agent 之前加一个路由层：白名单、频道分组、命令前缀、用户权限，都在这层完成。Agent 只接收通过路由的 envelope，输出统一 `OutboundAction`：

```ts
interface OutboundAction {
  type: 'send' | 'reply' | 'react' | 'edit';
  target: { platform: string; channelId: string; messageId?: string };
  content?: string;
  emoji?: string;
}
```

由对应 adapter 执行发送。

### 4. 入站队列与错误处理

每个平台单独维护入站队列，同一个 channel 串行处理，避免并发导致状态错乱。出站失败做有限重试，记录 ack 和错误原因。

## 踩坑点

- **消息长度不一致**：Telegram 单条约 4096，Discord 是 2000。AI 长回复需要统一分片，分段时保持顺序和引用关系。
- **Discord Intents**：必须在 Developer Portal 开启 `MessageContent` 和 `GuildMessages`，客户端也要配置对应 intents，否则 `messageCreate` 拿不到文本。
- **编辑/删除事件差异**：Telegram 的 `edited_message` 与 Discord 的 `messageUpdate` 字段不同，删除事件 Discord 更难可靠获取。不要强求完全一致，只做“尽力同步”。
- **ID 去重**：`messageId` 只在单平台内唯一，去重 key 必须用 `${platform}:${messageId}`。
- **附件时效**：Telegram 文件 URL 可能短期有效，Discord attachment URL 有签名过期。长期任务要么转存对象存储，要么只传递元数据。
- **限流独立**：两个平台限流策略不同，不要共用一个 rate limiter。Discord 有全局硬限制，Telegram 群聊对 bot 发送频率也敏感。

## 可复用建议

- Adapter 接口固定，新平台只需实现接口，不改 Agent core。
- 中间件管道统一为：`rate-limit -> auth -> route -> agent -> output`。
- 平台开关、路由规则放配置，不硬编码。
- 日志必须带 `platform/channel/messageId/latency`，多平台排障离不开这些字段。
- 先用白名单灰度，限制到单个用户或频道，验证稳定后再放开。

## 总结

跨平台消息路由不是“把两个平台的消息塞到一起”，而是把 platform 作为消息上下文的一部分。Agent 只处理统一 envelope 和 outbound action，adapter 只消化平台差异。这样后续接入其他平台，成本会低很多。

---

