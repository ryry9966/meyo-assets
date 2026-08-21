---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34090
source: 综合讨论
publishedAt: 2026-08-22
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

很多 Agent/自动化项目最初只接一个聊天平台，比如 Telegram。后来团队又需要 Discord 社区支持，如果直接再开一套 bot，很快会面临状态分裂、命令实现重复、MCP 工具调用逻辑不一致的问题。

真正要解决的不是“能收到两个平台的消息”，而是如何把平台差异隔离在边缘，让 Agent、MCP 工具和插件只处理一套统一语义。

## 做法

### 1. 定义统一消息信封

所有入站消息先被转成统一结构，再进入核心逻辑：

```ts
type Envelope = {
  platform: 'telegram' | 'discord'
  chatId: string
  userId: string
  messageId: string
  text: string
  replyTo?: string
  attachments?: { type: string; url?: string; name?: string }[]
}
```

这个信封不携带平台特定的原始对象，只保留回复需要的最小信息。

### 2. 连接器只做翻译和传输

Telegram connector 使用 `getUpdates` long polling 或 webhook；Discord connector 使用 gateway 事件。它们负责把平台消息转成 `Envelope`，并把 `update_id`、`message_reference` 等平台侧信息临时存下来，供出站适配器使用。

业务逻辑不要写在 connector 里，否则很快会变成两个平行代码库。

### 3. 路由和 Agent 核心

入站 `Envelope` 进入同一个队列，按 `platform + userId + chatId` 做会话索引。Agent 只看到统一文本、用户 ID、附件 URL、回复关系。命令解析也统一做，例如 `/status` 和 `!status` 都映射到同一个 handler。

MCP 工具接收标准参数，返回文本或结构化结果，不感知 Telegram 或 Discord。插件层同理，不做平台分支。

### 4. 出站适配器

Agent 产生通用回复：

```ts
{ target: Envelope, text: string, attachments?: [] , options?: {} }
```

Telegram 和 Discord 的 outbound adapter 各自负责 Markdown 转义、分片、上传附件、设置回复引用。这样格式差异不会污染核心逻辑。

## 踩坑点

- **格式差异**：Telegram 支持 MarkdownV2/HTML，Discord 是 Markdown 子集。不要在 Agent 里输出最终格式，应该输出语义化文本，由 adapter 转义。否则很容易出现 `_`、`*` 把消息炸掉。
- **长度限制**：Telegram 单条约 4096 字符，Discord 约 2000。长响应必须在 adapter 分片或转成文件。
- **Discord Intents**：Message Content Intent 必须在 Developer Portal 开启，否则 `message.content` 为空。修改 Intents 后需要重启并重新授权。
- **Telegram webhook 与 polling 冲突**：`setWebhook` 后 `getUpdates` 会返回 409。本地开发建议显式 `deleteWebhook` 后回退到 polling，生产环境再切 webhook。
- **回环与去重**：机器人自己发的消息也可能再次进入入站，需要过滤 author/bot；Telegram 按 `update_id` 去重，Discord 按 message id 去重，否则会出现“自己回复自己”或重复处理。
- **限流**：Discord 429 非常常见，需要对全局和 per-route 都做队列和指数退避；Telegram 也有约 30 msg/s 的限制，批量任务要注意。

## 可复用建议

- 将 connector 视为 IO adapter，核心 Agent 永远不直接 import Telegram/Discord SDK。测试时用 FakeConnector 跑完整路由即可。
- 会话隔离使用 `platform + userId + chatId` 组成 key，不要只按 userId。群聊、私聊需要分开。
- 命令前缀统一注册，并支持按 channel 配置启用范围，不要在 handler 里写死平台判断。
- MCP 工具尽量返回纯文本或标准结构化数据；附件传 URL，不要让工具直接上传到平台。
- 日志至少记录 `platform`、`message_id`、`route`、`latency`、`error`，跨平台排障会简单很多。

## 总结

一个 Agent 服务双端，本质不是写两个 bot，而是做一个消息总线加两个 adapter。平台差异放在边缘，核心只处理统一 Envelope。后续接入第三个平台，也只需要新增一个 connector 和一个 outbound adapter，而不是再复制一套业务逻辑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/bd3098d973da97f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/1697cd9ac03f63e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/5930923c1ee10656.png)

