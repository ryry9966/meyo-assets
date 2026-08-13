---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 32920
source: 综合讨论
publishedAt: 2026-08-13
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

很多 Agent 实践会先在一个平台跑通，比如 Telegram 或 Discord。但当用户分散在两个社区时，常见的做法是各起一套 Agent 实例、各配一份 prompt 和 memory。这带来几个问题：

- 同一套工具/插件要维护两份配置，环境变量开始漂移。
- 上下文与记忆互相割裂，同一个用户跨平台问同一件事，Agent 完全不记得。
- 核心逻辑里混入大量平台判断，后续接第三个平台越来越难。

本文记录一种工程化做法：用一个 Agent 逻辑，通过统一消息路由层同时服务 Telegram 和 Discord。重点不在“能收到消息”，而在如何隔离平台差异、控制会话边界、处理失败与重复投递。

## 问题

Telegram 和 Discord 的消息模型并不对齐：

- Telegram 有 chat_id、message_id、callback_query；群聊和私聊都通过 message 事件。
- Discord 分 message create/update/delete、interaction、reaction；频道 ID 与消息 ID 的幂等语义不同。
- 文本格式、附件、按钮、回复目标、限流规则都不一样。

如果让 Agent 直接感知这些平台对象，很快就会变成 `if telegram then ... else if discord then ...`。所以第一步不是接入，而是抽象。

## 做法

### 1. 定义统一 Envelope

让两个平台都转换成同一种内部消息结构：

```ts
type Envelope = {
  id: string;                 // platform + message_id 生成幂等键
  platform: "telegram" | "discord";
  chatId: string;             // 会话边界键
  senderId: string;
  senderName: string;
  text: string;
  attachments: { type: string; url?: string; fileId?: string }[];
  replyTo?: string;
  raw: unknown;               // 平台原始事件，仅 adapter 使用
};
```

Agent 只消费 `Envelope`，不接触 Telegram Bot API 或 Discord.js。

### 2. 平台 Adapter 各自收口

- Telegram Adapter：接收 webhook 或 long polling，把 `message` / `edited_message` / `callback_query` 转成 `Envelope`。
- Discord Adapter：接收 gateway 事件或 HTTP interaction，把 `messageCreate` / `interactionCreate` 转成 `Envelope`。

回复也通过统一出口：

```ts
sendReply(platform, target, content, options)
```

各 Adapter 再转换回平台能接受的形式。

### 3. 会话隔离

不要直接用全局 memory。会话键建议：

```ts
conversationKey = `${platform}:${chatId}`
```

Telegram 群聊和 Discord 频道天然属于不同会话，即使同一个用户跨平台说话，也不要把两个平台的上下文硬塞进同一 memory。否则 Agent 容易混淆“我刚才在 TG 群答应的事”和“现在 DC 频道的新需求”。

### 4. 路由与去重

入口只做三件事：转 Envelope、去重、交给 Agent。

去重 key 建议使用：

```ts
dedupeKey = `${platform}:${messageId}`
```

TTL 设 10 到 30 分钟。两个平台都可能因为 webhook 重试或 gateway 重连产生重复投递。

### 5. 失败回执与降级

Agent 处理完要回复时，`sendReply` 可能因为限流、附件过期、消息过长而失败。路由层应捕获错误，不要把 Discord 的 HTTP 400 直接抛给 Agent 主流程。可以选择：

- 文本降级：附件发送失败，改为发送文本摘要。
- 延迟重试：限流时退避重试。
- 失败记录：写入结构化日志，保留原始 Envelope ID。

## 踩坑点

### 1. 长任务不能同步等平台响应

Discord interaction 和 Telegram callback query 都有响应时限。Agent 如果做长时间工具调用，不能让用户点击按钮后一直等待。正确做法是先返回 ack，任务完成后再主动推送结果。

### 2. 消息长度与 Markdown 不一致

Discord 单条消息限制 2000 字符，Telegram 稍宽松，但双方 Markdown 解析不完全一样。建议 Agent 输出纯文本或受限 Markdown，Adapter 做平台适配，而不是直接把同一段字符串原样发两边。

### 3. 附件 URL 不能跨平台复用

Telegram 的 file_id 只有 Telegram 能用；Discord 附件 URL 通常有签名过期。如果真的需要把文件从 A 平台转到 B 平台，应当先下载到本地或对象存储，再上传到目标平台。否则过一会儿链接就失效。

### 4. 群聊触发策略要分开

Discord 频道里如果所有消息都进 Agent，噪声会很大；Telegram 群通常需要 @ 或命令触发。建议在 Adapter 层加触发策略：私聊默认全部响应，群聊按命令/ @ / allowlist 过滤。

### 5. Webhook 签名校验

Discord webhook/interaction 需要校验 Ed25519 签名；Telegram 建议设置 secret token。不要因为内网环境就裸奔，否则容易被人投递伪造消息。

### 6. 限流和重试

Discord 有全局 rate limit，Telegram 对 sendMessage 也有每秒限制。所有发送最好经过一个队列，统一排队和退避，避免 Agent 在高峰期触发平台封禁。

## 可复用建议

- **Adapter 不要做业务判断**。它只负责协议转换，不决定 Agent 说什么。
- **核心 Agent 不 import 任何平台 SDK**。这能保证第三方平台接入时不需要改动核心逻辑。
- **把 Envelope 作为测试边界**。单测时构造 Telegram 和 Discord 的 Envelope，核心逻辑用一套测试覆盖。
- **配置驱动接入**。新增平台时增加 adapter、token、触发规则，其余保持不变。
- **日志里带上 `platform` / `messageId` / `conversationKey` / latency**。跨平台问题没有这些字段很难排查。

## 总结

跨平台消息路由不是“同时登录两个账号”，而是一个工程边界问题。把平台差异关进 Adapter，把统一结构交给 Envelope，把会话隔离和失败处理做在路由层，一个 Agent 就能相对干净地服务 Telegram 和 Discord。

但也要接受一个现实：两个平台能力不会完全对齐。附件、按钮、消息格式、响应时限都不同。与其追求 100% 功能一致，不如提前设计好降级路径。能做到“核心 Agent 稳定、平台 Adapter 可替换、失败可追踪”，就已经比多实例分叉的维护方式好很多。

---

