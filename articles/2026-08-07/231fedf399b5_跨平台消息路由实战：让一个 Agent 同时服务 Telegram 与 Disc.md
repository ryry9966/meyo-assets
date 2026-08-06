---
title: 跨平台消息路由实战：让一个 Agent 同时服务 Telegram 与 Discord
feedId: 31922
source: 综合讨论
publishedAt: 2026-08-07
---

在 OpenClaw 社区，我们常常把一个 Agent 部署到单个平台——最常见的是 Discord 或 Telegram。但当你的用户分布在多个平台，或者你希望运维、通知、问答都在同一个 Agent 上完成时，就必然面对“一个大脑，多双耳朵”的问题：如何让同一个 Agent 同时可靠地响应来自 Telegram 和 Discord 的消息，并且保持上下文一致？

这篇文章记录我在一个内部助手项目中的实践，涵盖从架构选型、消息适配、状态贯通，到实际踩坑和可复用建议的完整过程。技术栈以 OpenClaw Agent 为中心，配合轻量 MCP 适配层实现跨平台路由。

## 一、背景：两个平台，两个世界

Telegram 和 Discord 虽然都是聊天平台，但在消息模型上差异巨大：

- **消息结构**：Telegram 的消息是扁平文本 + 实体标注；Discord 的 Message 包含 embed、component、interaction 等多种富结构。
- **身份体系**：Telegram 用 user_id/chat_id；Discord 用 user ID/guild ID/channel ID，且存在 Webhook 与应用身份的区别。
- **交互模式**：Discord 原生支持斜线命令、按钮、下拉菜单；Telegram 依赖 bot command + inline keyboard。
- **速率限制与重试策略**：两套 API 的限流逻辑和错误处理完全不同。

如果直接在 Agent 逻辑里分别调用 Telegram Bot API 和 Discord.js/discord.py，代码会迅速膨胀成难以维护的适配层。我们需要一种方式让 Agent 只处理“意图”，而把平台差异交给专门的路由层。

## 二、问题抽象：统一消息通道

核心思路是**建立一个平台无关的消息信封**。任何平台的消息进入系统后，被适配器转换成标准格式 `Envelope`：

```json
{
  "platform": "telegram",
  "channel_id": "tg:-123456",
  "user_id": "tg:987654",
  "thread_id": "tg:987654",
  "text": "/status",
  "attachments": [],
  "raw": { ... }
}
```

OpenClaw Agent 接收 `Envelope`，处理后返回 `Reply` 对象（同样平台无关），然后由对应平台的发送适配器转换回平台原生格式并传出。

这样 Agent 核心就完全无状态、无平台依赖，仅依赖 `user_id` 和 `thread_id` 做会话管理。这也为未来接入 WhatsApp、Slack 等平台留了扩展点。

## 三、具体实现步骤

### 1. 平台适配器

使用独立的轻量服务（可以是同一进程内的模块）对接各平台 SDK：

- **Telegram 适配器**：基于 `python-telegram-bot`，在 `on_message` 回调中构造 `Envelope`，发送到统一消息队列。
- **Discord 适配器**：基于 `discord.py`，监听 `on_message` 与 `on_interaction`，统一映射为 `Envelope`。对于 slash command，`text` 字段填完整命令字符串；对于按钮点击，携带 `custom_id` 并在 `raw` 中保留上下文。

两个适配器均以 `asyncio` 协程运行，通过内部 Redis pub/sub 或直接函数调用与 Agent 通信。避免在适配器里做任何业务决策。

### 2. Agent 核心

OpenClaw Agent 暴露一个 `process(envelope: Envelope) -> Reply` 接口。处理流程：

1. **身份识别**：根据 `platform + user_id` 查用户画像（权限、偏好语言等）。
2. **会话恢复**：以 `thread_id` 为键拉取最近 N 轮对话，构造 prompt 上下文。
3. **意图路由**：交给 OpenClaw 的 Planner 或直接 LLM 调用生成回复。回复可以是纯文本或 Markdown。
4. **输出标准化**：返回统一的 `Reply`，包含 `text` 和可选的 `components`（按钮、快捷回复等标准化描述）。

### 3. 消息队列与会话存储

为了避免单点阻塞和方便横向扩展，我在适配器与 Agent 之间加入了一层轻量队列（本地 Redis Streams）。Telegram/Discord 适配器将 `Envelope` 写入队列 `agent:inbox`，Agent 作为消费者逐条处理。

会话上下文存储在 Redis 中，`key = session:{platform}:{thread_id}`，使用 List 结构保存最近 20 条对话。每次 Agent 处理完一条消息，更新队列并设置 TTL。

> 踩坑点：不同平台 `thread_id` 的生成方式。Telegram 私聊中 `chat_id` 直接作为 `thread_id`；群聊中如果 bot 未开启 Privacy Mode，则用 `chat_id` 无法区分不同用户话题。我最终使用 `chat_id:user_id` 组合作为 thread_id，确保同一群内不同用户的会话隔离。Discord 中用 `channel_id:user_id`，并需要小心处理 DM 通道（`channel_id` 可能重复？实际 Discord DM 的 channel_id 唯一，因此直接使用 `channel_id` 即可隔离）。

### 4. 输出适配与重试

Agent 返回 `Reply` 后，Dispatcher 根据 `reply.platform` 把消息交给对应适配器执行发送。这里要特别注意：

- **Telegram** 支持 MarkdownV2/HTML parse mode，需要转义特殊字符。OpenClaw 生成的 Markdown 需要先转换成 Telegram 兼容格式。实践中我写了一个简单的转换函数，处理 `**bold**`、`_italic_`、code block 等常见格式，其余忽略。
- **Discord** 的消息有 2000 字符限制，超过需要分片。对于长回复，在 Discord 适配器中自动按段落切断，发送多条消息并在最后一条附加提示（如“（续上）”）。Telegram 的消息限制为 4096 字符，但也建议在接近时长话短说。

重试策略：两个平台的 API 都可能因限流返回 429。我使用指数退避重试最多 3 次，并将失败消息记录到死信队列，便于人工排查。

## 四、实际踩坑记录

1. **Telegram callback query 的时效性**：按钮回调必须在 20 秒内应答，否则会显示时钟超时。如果在 Agent 处理流程中耗时过长，需要先发送一个空响应 `answer_callback_query` 通知 Telegram 已收到，然后再异步更新消息。我的做法是适配器收到按钮 `Envelope` 后，立即应答空 callback，同时将处理交给 Agent，结果通过 `edit_message_text` 回写。

2. **Discord 的交互 token**：Interaction 的回执需要在 3 秒内给出首次响应，同样需要异步处理。使用 `defer` 延迟回复，然后通过 `followup` 发送最终结果。适配器必须正确管理 interaction token 的生命周期。

3. **用户 ID 冲突**：不同平台的用户 ID 可能相同（例如 Telegram 的 `123456` 和 Discord 的 `123456`），所以在 `user_id` 中统一加了前缀 `tg:` 和 `dc:`，避免会话或权限错乱。

4. **富文本组件差异**：Telegram inline keyboard 和 Discord 的 buttons/select menu 数据格式完全不同，但行为类似。我的做法是在 `Reply` 中定义了一个简化的 `ActionRow` 结构（`type, label, value`），由各平台适配器转换成原生组件。这损失了一部分原生特性，但保持了 Agent 逻辑简洁。

## 五、可复用建议

- **始终做平台前缀隔离**：所有 ID、会话键、权限角色，都加上平台标识符，避免未来多平台接入时发生隐式冲突。
- **优先让 Agent 输出纯文本/Markdown**，组件交互尽量用标准化元语，由适配层实现平台转换。不要试图让 Agent 生成平台特定的 JSON。
- **监控消息处理延迟**：从入队到发送成功记录全链路时间。Telegram 和 Discord 用户对响应延迟敏感，长耗时任务（如 RAG 检索）可以先用 typing indicator 安抚，然后更新。
- **错误隔离**：一个平台的发送失败不应该影响另一个平台的消息处理。确保 Dispatcher 中异步发送，并使用 try/catch 包裹每个平台的发送调用。
- **逐步开启组件交互**：一开始只做文本问答，跑稳定后再加入按钮。复杂的交互（如模态框、下拉菜单）建议等文本流程成熟后再扩展，避免一次性面对太多平台差异。

## 六、总结

让一个 Agent 同时服务 Telegram 和 Discord 并不需要魔改核心逻辑，关键是建设稳固的平台适配层和统一的消息信封。通过标准化 `Envelope` 和 `Reply`，Agent 获得了一个干净的“纯文本大脑”，而各平台的复杂细节被封装在适配器内。

这套做法在我自己的小规模场景中稳定运行了数周，日均处理消息约 2000 条，跨平台延迟中位数小于 1.2 秒。如果你也在 OpenClaw 上做多平台部署，希望这篇文章能帮你少走一些弯路。欢迎在社区分享你的跨平台方案，一起打磨出更通用的实践。

---

