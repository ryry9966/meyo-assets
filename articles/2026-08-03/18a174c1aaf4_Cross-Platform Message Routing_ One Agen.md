---
title: Cross-Platform Message Routing: One Agent for Both Telegram and Discord
feedId: 31397
source: 综合讨论
publishedAt: 2026-08-03
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

在日常的 OpenClaw 实践中，我们经常会为一个自动化场景专门搭建一个 Agent。当社区同时活跃在 Telegram 和 Discord 上时，相同的接入逻辑——例如工单查询、状态监控、知识库问答——往往需要重复部署两套完全独立的 Agent 实例，分别对接各自的 Bot API。维护两套配置、两套安全校验与两套消息处理流程，不仅增加了运维负担，还容易导致行为不一致。

实际上，这两个平台的消息本质上都是“来自用户的一段文本，期望得到一段回复”。真正的难点在于平台特有的消息格式、交互原语（按钮、slash命令、嵌入卡片）以及不同的并发和连接模型。统一抽象这些差异后，完全可以让一个 Agent 核心同时驱动两个平台的 Bot。

## 核心问题

我们要解决三个层面的问题：

1. **协议适配层**：Telegram 使用长轮询（getUpdates）或 Webhook + JSON，Discord 通过 WebSocket Gateway 或 Webhook 交互。消息体结构完全不同。
2. **消息归一化**：需要将不同平台的交互抽象为统一的 `UserMessage` 对象，包括发送者 ID、平台标识、会话 ID、消息正文和可能的上下文附件。
3. **回复分发**：Agent 生成回复后，根据消息来源平台，将回复转换回对应平台的格式（Telegram 支持 MarkdownV2/HTML，Discord 使用 Embed），并通过该平台的 API 发送回去。

## 实践步骤

这里基于 OpenClaw 的插件体系与 MCP 工具调用，搭建一个同时服务 Telegram 和 Discord 的 Agent。

### 1. 创建统一的消息模型

```python
from dataclasses import dataclass
from typing import Optional, Literal

Platform = Literal["telegram", "discord"]

@dataclass
class NormalizedMessage:
    platform: Platform
    user_id: str
    chat_id: str       # 对 Discord 可能是 channel_id+DM 的组合键
    text: str
    message_id: str    # 用于后续回复/编辑
    attachments: Optional[list] = None
```

每次收到平台原始消息，先转换成 `NormalizedMessage`，再交给 Agent。

### 2. 平台接入适配器

为每个平台编写适配器，内部维护平台 SDK 客户端（如 python-telegram-bot 或 discord.py）。核心职责：接收原始事件、转换为归一化消息、提供发送回复的方法。

Telegram 适配器示例片段（使用 long polling）：

```python
async def telegram_polling(agent_callback):
    async with TelegramClient(...) as client:
        @client.on_message()
        async def handler(event):
            msg = NormalizedMessage(
                platform="telegram",
                user_id=str(event.sender_id),
                chat_id=str(event.chat_id),
                text=event.text,
                message_id=str(event.id)
            )
            reply = await agent_callback(msg)
            await client.send_message(event.chat_id, reply.text, parse_mode="MarkdownV2")
```

Discord 适配器类似，但要处理 Gateway 事件：

```python
@bot.event
async def on_message(message):
    if message.author.bot:
        return
    msg = NormalizedMessage(
        platform="discord",
        user_id=str(message.author.id),
        chat_id=str(message.channel.id),
        text=message.content,
        message_id=str(message.id)
    )
    reply = await agent_callback(msg)
    await message.channel.send(reply.text)  # 复杂回复可用 embed
```

### 3. Agent 核心逻辑

Agent 接收 `NormalizedMessage`，可以基于 OpenClaw 的对话树、MCP 工具或外部 API 生成回复。多平台共享同一套上下文管理——通过 `platform + user_id` 作为会话键，使同一个用户在两个平台上的对话独立，但不影响 Agent 逻辑的复用。

如果需要平台相关的特殊回复（例如 Discord 要求使用按钮组件），可在 Agent 的回复模型中增加一个 `actions` 字段，由适配器负责渲染。

## 踩坑点

- **消息长度限制**：Telegram 文本消息最长 4096 字符，Discord 2000 字符。Agent 回复较长时必须分段。统一处理的建议是 Agent 生成回复时主动按 1900 字符分段，然后由适配器用多条消息发送。
- **交互组件差异**：Telegram 的 InlineKeyboard 和 Discord 的 MessageComponent 结构不同。如果 Agent 需要提供按钮、下拉选择等 UI，建议在归一化层抽象为通用动作类型（例如 `action_type: url_button`），再由适配器映射为各自平台的组件代码。
- **并发与消息顺序**：Discord 的消息可能高度并发，Telegram 串行更新更有序。Agent 内部如果是有状态的任务（例如多轮澄清），需要确保平台 + 用户维度的串行处理，可用 asyncio.Queue 按会话键分发。
- **平台特定限制**：Telegram 群组中 Bot 需要被 @ 或者使用命令才能触发，Discord 默认监听所有消息。要小心隐私和 spam 风险，可在归一化消息中加一个 `should_respond` 的标志位由适配器判断。
- **认证与安全**：两个平台的 Token 和安全校验方式不同。不要复用 Webhook 端点，而是各自独立监听，将 Token 或 secret 校验放在适配器层。Agent 核心不需要关心平台身份。

## 可复用建议

1. **抽象统一消息通道**：如果你计划支持第三个平台（如 Slack、Matrix），只需再实现一个适配器，Agent 核心零改动。建议将平台接口定义为抽象基类，强制实现 `normalize` 和 `send_reply` 方法。
2. **配置隔离**：使用环境变量或配置平台指定 Bot Token、intents 等，每个适配器独立初始化，避免互相污染。
3. **错误处理与重试**：平台 API 调用可能因 rate limit 或网络问题失败。在适配器内部实现指数退避重试，而不是在 Agent 逻辑中处理。
4. **多会话键**：使用 `platform + chat_id` 作为会话唯一标识，确保同一个人在群聊和私聊的会话也是隔离的，符合直觉。
5. **监控与日志**：在归一化消息中注入平台信息，日志里可以轻松区分问题来源，便于排障。推荐通过结构化日志记录每条消息的处理耗时和状态。

## 总结

让同一个 Agent 同时服务 Telegram 和 Discord，本质上是在平台 API 和 Agent 核心之间引入一层薄薄的适配抽象。只要把平台特有的消息格式、交互方式、发送限制封装在适配器内部，Agent 就可以专注于“理解与生成”。这种模式不仅在工程上减少了重复代码，还为未来扩展更多平台提供了灵活的基础。在实际生产环境中，配合 OpenClaw 的任务调度与 MCP 工具调用，一套 Agent 即可为多个社区提供一致的自动化体验，而不增加运维的复杂度。

---

