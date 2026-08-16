---
title: 跨平台消息路由：一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 33521
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

很多 OpenClaw 实践最初都从单一 IM 开始：先接 Telegram，跑通 Agent 后再接 Discord。问题在于，如果直接按平台写业务逻辑，很快会出现两套命令、两套状态、两套权限，后续维护成本翻倍。

更合理的目标是：**Agent 核心只处理统一消息，平台差异放在边缘适配器里消化。** 这样同一个 OpenClaw Agent 可以同时服务 Telegram 和 Discord，行为一致，且后续加 Slack、Matrix 或 Web 时不需要改核心逻辑。

## 问题

Telegram 和 Discord 的消息模型差异很大：

- Telegram 有 `message`、`callback_query`、`edited_message`，会话由 `chat_id` 决定。
- Discord 有 `message`、`interaction`、`thread`，还有 guild/channel 层级。
- 回复语义不同：Telegram 是 `reply_to_message_id`，Discord 是 `message_reference` 或直接用 thread。
- 消息长度不同：Discord 普通消息约 2000 字符，Telegram 约 4096。
- Markdown 解析规则不同，文件、限流、ACK 机制也完全两套。

如果不用统一路由层，业务代码里会散落大量 `if platform === 'telegram'` 判断。

## 做法

### 1. 定义统一入站事件

先在 OpenClaw 插件目录下实现两个 adapter，它们只做一件事：把平台消息转换为统一事件。

```ts
type InboundEvent = {
  platform: 'telegram' | 'discord';
  chatId: string;
  userId: string;
  messageId: string;
  threadId?: string;
  text: string;
  attachments: Attachment[];
  replyTo?: string;
  raw: unknown;
}
```

`chatId` 统一为字符串，避免 Telegram 数字 ID 和 Discord snowflake 混用导致类型问题。`replyTo` 保留原始回复目标，后续多轮上下文需要用到。

### 2. 用会话键隔离状态

路由层生成会话键：

```ts
const sessionKey = `${event.platform}:${event.chatId}${event.threadId ? ':' + event.threadId : ''}`;
```

不能用 `userId` 作为唯一会话键，否则同一个用户在两个平台会被当成同一个人，权限和上下文都会串。会话状态按这个 key 分片存储。

### 3. 统一出站消息

Agent 处理完后不直接调平台 API，而是返回统一的 `OutboundMessage`：

```ts
type OutboundMessage = {
  text: string;
  replyTo?: string;
  chunks?: string[]; // 由 core 预先分段
}
```

adapter 的 denormalize 层负责渲染：Telegram 走 `sendMessage` + `reply_to_message_id`，Discord 走 `reply` 或 `channel.send`。这样核心逻辑完全不知道平台细节。

### 4. 命令与权限映射

命令表里不要硬编码 `/status`，而是维护平台别名：

```yaml
commands:
  status:
    telegram: /status
    discord: /status
  help:
    telegram: /help
    discord: !help
```

权限判断也要按平台隔离。Discord 的用户 ID 和 Telegram 用户 ID 没有关系，不能直接复用白名单。通常会建立 `platform:userId` 形式的身份映射。

### 5. 媒体先落盘

平台对文件大小和类型限制不一致。建议 adapter 收到附件后先下载到对象存储或本地临时目录，然后给 Agent 一个统一的 `Attachment` 引用。出站时根据平台限制决定是直接上传还是发送 URL。

## 踩坑点

### 长度不要直接截断

Discord 会拒绝超过 2000 字符的消息，Telegram 虽然更长，但直接截断可能破坏 Markdown 或代码块。最好在 core 层按语义分段，adapter 再根据平台限制合并或拆分。

### 及时 ACK

Discord 的 interaction 要求 3 秒内响应，否则按钮、斜杠命令会超时。Telegram 的 callback query 也需要 `answerCallbackQuery`。如果 Agent 处理耗时较长，adapter 应先返回一个“处理中”占位，完成后再编辑消息或新发消息。

### 重连与事件顺序

Discord 网关断线后要使用 resume，否则会丢事件。Telegram `getUpdates` 的 `offset` 必须在处理成功后持久化，否则重启会重复消费或漏消息。两者都应该做幂等，用 `messageId` 去重。

### ID 变化

Telegram 群组升级到 supergroup 时 `chat_id` 会变，历史上很多会话状态会失效。Discord 的 channel/thread ID 相对稳定，但 guild 可能因为 bot 被移除而不可达。适配器要能容错，不能让一个平台的事件拖垮另一个平台。

### 不要共用 Markdown 渲染器

Telegram MarkdownV2 对特殊字符很敏感，Discord 规则又不同。不要试图写一个通用的 Markdown 渲染器，应该在各自 adapter 里单独处理。

## 可复用建议

- **适配器只做翻译**：不要在里面保存业务状态，状态全部进会话层。
- **会话键分片**：统一用 `platform:chatId[:threadId]`，日志和指标也带上 platform tag。
- **配置分离**：Token、webhook secret、guild id 等放环境变量或独立配置文件。
- **先文本，后媒体**：第一版只支持文本消息和基础回复，跑通后再加文件、按钮、斜杠命令。
- **提供 echo/dry-run 模式**：测试时让 Agent 原样返回规范化事件，方便排查平台差异。

## 总结

跨平台消息路由的本质是边界管理：让 Agent 核心只处理统一事件，所有平台差异都沉淀在边缘 adapter 中。这样无论同时接多少个平台，业务逻辑都不会被平台模型污染。OpenClaw 的插件机制很适合做这件事，关键是前期把入站事件、会话键和出站消息模型定义清楚。

---

