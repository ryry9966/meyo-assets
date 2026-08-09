---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 32209
source: 综合讨论
publishedAt: 2026-08-09
---

在 Agent 实践中，常会遇到这样一个需求：同一个对话核心，需要同时接住来自 Telegram 群组和 Discord 频道的消息，并把回复准确投递回去。如果为每个平台重复造轮子，不仅维护成本高，且行为一致性很难保证。本文记录一次基于统一消息模型的跨平台路由实践，面向已经接触过 OpenClaw、MCP 或自动化工具的技术用户，包含设计、踩坑和可复用方案。

## 背景与问题

Agent 要服务的用户分布在多个即时通信工具，最常见的就是 Telegram 和 Discord。两个平台的 Bot API 差异显著：Telegram 使用 HTTP Webhook 推送 `Update`，Discord 则可以选择 Gateway（WebSocket）或 HTTP Interactions（Webhook）。消息结构、字段名、媒体附件处理方式也完全不同。如果直接在 Agent 逻辑里分别处理这两套协议，代码很快就会变得难以维护。更麻烦的是，两个平台对响应时间、重试策略、速率限制的要求不一致，任意一点疏忽都可能导致消息丢失或触发平台限流。

因此，一个可落地的方案是：**在 Agent 与平台之间插入一个路由层，将不同平台的消息统一为内部消息格式，Agent 只消费这一种格式；处理完成后，路由层再将回复转换回各平台的 API 调用。** 这样，Agent 只需要理解“用户说了什么，在哪个会话里”，完全不关心来源是 Telegram 还是 Discord。

## 设计方案与实现步骤

### 1. 统一消息模型

首先，定义一套与平台无关的消息结构，例如：

```json
{
  "platform": "telegram" | "discord",
  "chat_id": "tg:chat:123 | dc:channel:456",
  "user_id": "tg:user:789 | dc:user:101",
  "text": "hello",
  "attachments": [
    {"type": "image", "url": "https://..."}
  ],
  "reply_to_id": "msg:xxx"
}
```

这个模型覆盖了文本消息和简单附件，也可以通过 `reply_to_id` 标记对话上下文。Agent 的输出则是同样的结构，只是 `text` 变为回复内容。

### 2. 路由服务实现

路由服务用 FastAPI 实现，暴露两个端点：

- `POST /webhook/telegram`  
- `POST /webhook/discord`

Telegram 一端的处理比较直接：收到 `Update` 后立即返回 200，以保证 Webhook 不会因处理慢而被重试。我们将原始 `Update` 推入一个本地内存队列（生产环境建议用 Redis Stream），由后台 worker 异步消费。Worker 将 `Update` 中的 `message` 字段转换为统一消息格式，并携带一份 `reply_meta`（用于后续将回复发回来源平台，包含需要的 `chat_id`、`bot_token` 等）。

Discord 端更复杂一些。如果采用 HTTP Interactions，每次用户执行斜杠命令或点击按钮，Discord 会向我们的端点发送 `POST`，且**必须在 3 秒内返回一个初始响应，否则交互会失败**。因此，我们同样立即返回一个 `type: 5` 的 `DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE`，告知 Discord“正在处理，稍后回复”。之后通过 `interaction token` 和 `application_id` 发送后续消息（followup）。这个 deferred 消息的发送也通过后台 worker 异步进行。转换过程与 Telegram 类似，但需要处理 interaction 的 `data` 字段，从中提取用户输入或自定义 ID。

### 3. Agent 与队列解耦

Worker 将统一消息推入队列，Agent 作为一个独立消费者拉取消息。Agent 内部可以调用 MCP 工具，比如知识库检索、外部 API 等，生成回复。回复同样被推入一个“出站队列”，由另一个 worker 负责回传。

回传时，通过统一消息中的 `platform` 字段选择适配器：

- **Telegram 适配器**：调用 `sendMessage` API，注意消息长度超出 4096 字符时分段发送；附件先上传，获得 `file_id` 再发送。
- **Discord 适配器**：使用 `POST /webhooks/{app_id}/{token}` 发送 followup 消息，消息长度超过 2000 字符也需分段，支持 embed 或直接上传文件（`attachment://`）。

### 4. 速率限制与重试

Discord Webhook 有全局速率限制，特别是同一频道内短时间内大量发送消息会导致 429。我们在适配器中实现了基于响应的动态延时和指数退避重试。Telegram 的 API 相对宽松，但也需要处理偶发的 429，重试逻辑类似。

## 踩坑点与注意事项

- **Webhook 签权**：Discord 要求在 `X-Signature-Ed25519` 和 `X-Signature-Timestamp` 中验证请求签名，避免伪造请求。必须严格校验。Telegram 可通过 `secret_token` 在 `setWebhook` 时设置，配合请求头 `x-telegram-bot-api-secret-token` 校验。
- **消息顺序**：如果使用异步处理，两个平台的同一个会话内消息可能乱序。对于大多数对话场景，这影响不大，但如果 Agent 依赖严格顺序，可以按会话 ID 分区，保证同一会话消息顺序处理。
- **附件差异**：Telegram 的图片可以直接通过 URL 发送，Discord 更倾向于本地附件或 CDN 链接。跨平台时，若用户通过 Telegram 发送图片，Agent 想在 Discord 回复同一图片，需要先下载再上传到 Discord，注意文件大小限制（Discord 普通 bot 10 MB）。
- **用户身份映射**：如果希望跨平台保持用户上下文，需要建立身份映射表。比如根据用户 ID 或用户名，绑定两个平台上的身份。可用一个小表持久化。
- **错误恢复**：队列 worker 如果崩溃，消息可能丢失。建议使用持久化队列（如 Redis Stream 的消费者组）并开启消息确认机制。发送失败的消息可以写入死信队列报警。

## 可复用架构与建议

整个方案可以用一张图概括：**平台 Webhook → 路由层 → 入站队列 → Agent（+ MCP 工具）→ 出站队列 → 平台适配器 → API**。

这种分离的好处是：
- 新增平台只需实现新的适配器，不改动 Agent 逻辑。
- Agent 开发者只需关注统一消息格式，大幅降低平台耦合。
- 队列解耦天然支持横向扩展、错误重试和降级。

如果你的 Agent 已经大量使用 MCP，甚至可以考虑将“平台适配器”也封装成一个 MCP server，提供 `send_message`、`upload_file` 等工具，让 Agent 通过工具调用完成回复。但要注意这样会引入额外的延迟，适合对实时性要求不高的场景。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，本质上是一次“协议适配+消息路由”的工程实践。核心思路是构建统一消息模型，用异步队列隔离平台特性，并针对各平台的超时、签名、速率限制做细致处理。这样的结构可以很自然地扩展到更多 IM 平台，也能让 Agent 自身的迭代完全不受外部通信协议变更的影响。

如果你的场景中平台数量不止两个，或者有更复杂的媒体交互需求，建议从一开始就设计好扩展点：平台适配器作为插件加载，统一消息模型预留 metadata 字段，避免后期大规模重构。

---

