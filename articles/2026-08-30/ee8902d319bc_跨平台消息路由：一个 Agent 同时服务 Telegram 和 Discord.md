---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 35330
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在自动化实践里，常常需要让同一个 Agent 同时出现在 Telegram 和 Discord 两个群里，处理相似的任务：接收指令、查询状态、触发操作、回复结果。如果为每个平台单独写一套 bot 逻辑，维护成本翻倍，而且 Agent 的上下文和状态容易割裂。OpenClaw 的插件化 adapter 机制可以把平台接入和核心逻辑分开，让一个 Agent 实例统一处理多平台消息。

## 问题

Telegram 和 Discord 的 API 模型差异明显。Telegram 用 `chat_id + message_id` 定位对话，消息实体是数组，需要手动解析命令和参数。Discord 用 `channel_id + message_id`，命令通常靠前缀，还需要开启 Message Content Intent 才能读取用户消息内容。直接在一个 handler 里写 `if platform == ...` 会导致代码越来越乱。

## 做法 / 步骤

**1. 定义统一消息模型**

在 OpenClaw 插件里，先抽象出两个结构：

```python
class InboundMessage:
    platform: str
    chat_id: str
    sender_id: str
    text: str
    mentions: list
    reply_to: str | None
    raw: dict

class OutboundMessage:
    platform: str
    chat_id: str
    text: str
    reply_to: str | None
    options: dict
```

平台适配器负责把各自的原始事件转成这个结构，核心 Agent 只处理这个结构。

**2. 写 Telegram 适配器**

使用 `python-telegram-bot` 或 `aiogram`，监听 update，转换消息。注意处理 `entities`：如果消息包含 `bot_mention` 或 `command` entity，提取命令和参数。将 `chat_id` 映射为统一的 `conversation_id`。

**3. 写 Discord 适配器**

使用 `discord.py`，确保启用 `message_content` intent。在 `on_message` 里过滤 bot 自己，提取 `content`，去掉命令前缀（例如 `!` 或 `/`）。`channel_id` 作为 `conversation_id`。

**4. 注册到 OpenClaw Agent**

两个适配器作为插件加载，每个适配器将 `InboundMessage` 投递到同一个消息队列或直接调用 `agent.handle(inbound)`。Agent 处理后返回 `OutboundMessage`，适配器根据 `platform` 字段调用对应平台的发送 API。

**5. 配置与启动**

环境变量配置两个平台的 token，启动时加载插件：

```
OPENCLAW_ADAPTERS=telegram,discord
TELEGRAM_TOKEN=...
DISCORD_TOKEN=...
```

同一个 Agent 进程同时监听两个平台。

## 踩坑点

- **Telegram 的 entities 解析**：如果简单取 `message.text`，命令可能带 `@botusername`，需要 strip 掉，否则路由不到。建议使用 bot 库自带的 command 处理或手动解析 entity offset。
- **Discord 的 Intents**：忘记在 Developer Portal 开启 Message Content Intent，或者代码里没设置 `intents.message_content = True`，会导致收不到任何消息，而且不会报错，只看到 bot 在线但静默。
- **消息长度限制**：Telegram 上限约 4096 字符，Discord 2000 字符。Agent 返回长文本时需要分段，适配器里根据平台限制拆分，避免发送失败。
- **用户身份映射**：同一个真实用户在两个平台的 ID 不同，如果 Agent 需要记住用户偏好，要么明确按平台隔离，要么维护映射表。
- **限流和重试**：Telegram 对发送频率有限制（约 30 msg/s 每 chat，全局更严格），Discord 也有 rate limit。发送失败时要指数退避，不要立即重试。
- **事件回环**：如果 Agent 自己有消息发送到群里，适配器要过滤掉自己发送的消息，否则可能触发无限循环。在 Discord 里检查 `message.author.bot`，Telegram 里检查 `message.from.is_bot`。

## 可复用建议

- 适配器保持无状态，只做转换和发送。会话状态放在 Agent 核心，用 `conversation_id` 区分。
- 引入简单的消息队列（如 `asyncio.Queue`）解耦接收和处理，避免某个平台响应慢拖累另一个。
- 日志里统一带 `platform` 和 `message_id`，方便排障。
- 写一个 mock adapter 做单元测试，模拟两个平台的消息流，验证路由逻辑。
- 对平台特定功能（如 Telegram 的 inline keyboard、Discord 的 embed）通过 `OutboundMessage.options` 传递，核心不关心具体格式。

## 总结

一个 Agent 同时服务多个平台的本质是抽象统一消息模型 + 适配器解耦。OpenClaw 的插件机制适合做这件事。先把消息模型定清楚，再分别实现适配器，踩坑主要集中在上行解析和下行限制。做好异步、日志和限流，这套结构可以稳定运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f488f976d49de93c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/be34b0bc9fb73723.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c17e5cd1e217bab0.png)

