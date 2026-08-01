---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 31176
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

用 OpenClaw 构建 Agent 时，常见的需求是把同一个机器人部署到多个聊天平台。开发者通常会经历这两个阶段：

1. **快速验证期**：为 Telegram 写一套逻辑，Discord 再复制一份，勉强跑通。
2. **维护噩梦期**：对话逻辑、工具调用、上下文管理散落在两个项目里，改一行 Prompt 要改两个仓库，状态不同步，排错时得切来切去。

最终大家都会想：能不能让 **一个 Agent 核心同时处理来自 Telegram 和 Discord 的消息**？不是两个独立实例，而是同一个进程、同一套记忆、同一套工具链。

## 问题分析

这件事的难点不在 Agent 本身，而在「消息如何进出」。两个平台的差异主要体现在：

- **消息结构不同**：Telegram 是 `message.text` + `chat_id`，Discord 是 `Message.content` + `channel_id`；Telegram 有 inline query / callback query，Discord 有 slash command / modal。
- **用户身份模型不同**：Telegram 的 `user_id` 是跨群组的全局整数；Discord 则是 `(guild_id, user_id)` 的组合，同一个人在不同服务器里身份不同。
- **媒体与交互限制**：Discord 支持富文本嵌入（embed），Telegram 不支持等同形态；Telegram 消息长度上限约 4096 字符，Discord 虽然单条也有限制但可发多条。
- **协议与并发模型**：Telegram 通过长轮询或 Webhook 以 HTTP 为中心；Discord 通过 WebSocket 保持长连接。在同一个进程里同时维护两种连接时，并发控制与错误恢复会变得棘手。

如果我们强行让 Agent 直接理解平台差异，Agent 的 codebase 会迅速腐化。更好的做法是**引入一层消息适配**。

## 核心思路

将架构拆成三层：

```
Platform Adapters  →  Message Router  →  Agent Core
(Telegram/Discord)     (统一消息格式)     (OpenClaw Agent)
```

- **Platform Adapter**：负责翻译平台 API 与统一消息格式之间的一切差异，包括连接管理。
- **Message Router**：接收统一消息，决定回复目标（可能存在多平台同一用户的场景，比如用户从 Telegram 切到 Discord 继续对话，这属于进阶需求，本文不展开）。
- **Agent Core**：只处理 `UserMessage`（含用户输入 + session_id + 平台来源标记），返回 `AgentReply`（文本 + 可选附件），不感知具体平台。

这样，新增一个平台只需要实现新的 Adapter，Agent 内核一行不用改。

## 具体实现步骤

### 1. 定义统一消息格式

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class PlatformMessage:
    platform: str          # "telegram" or "discord"
    session_id: str        # 全局唯一的会话 ID
    user_id: str           # 平台内的用户 ID
    text: str
    reply_to_platform_id: Optional[str] = None  # 用于定位回复给谁
```

`session_id` 的生成规则很重要。为了避免两个平台的用户 ID 碰撞，采用“平台前缀 + 用户 ID + 频道/群组”组合：

```python
# Telegram 私聊：tg:private:123456
# Telegram 群组：tg:group:-100123456:user_7890
# Discord 频道：disc:guild_xxx:channel_yyy:user_zzz
```

### 2. 实现 Telegram Adapter

使用 `python-telegram-bot`，通过 `Application` 构建长轮询或 Webhook 服务。关键代码片段：

```python
from telegram.ext import Application, MessageHandler, filters

async def handle_telegram(update, context):
    msg = update.effective_message
    session_id = f"tg:private:{msg.chat_id}"  # 简化示例
    plat_msg = PlatformMessage(
        platform="telegram",
        session_id=session_id,
        user_id=str(msg.from_user.id),
        text=msg.text,
        reply_to_platform_id=str(msg.message_id),
    )
    reply = await agent_core.process(plat_msg)
    await msg.reply_text(reply.text)
```

**踩坑点：**
- Telegram 群组消息默认不带 `/` 命令不会进 handler，需要设置 `filters.TEXT & ~filters.COMMAND` 或使用 `filters.ALL`。
- Callback query 是一种独立事件，无法直接 `reply_text`，需要用 `callback_query.edit_message_text` 或 `callback_query.answer`。
- 文件发送需要使用 `context.bot.send_document` 而非 `msg.reply_document`，以免 reply 绑定错误。

### 3. 实现 Discord Adapter

使用 `discord.py`，需要启用 `message_content` Intent。

```python
import discord

class DiscordBot(discord.Client):
    async def on_ready(self):
        print(f'Logged in as {self.user}')

    async def on_message(self, message):
        if message.author == self.user:
            return
        session_id = f"disc:{message.guild.id}:{message.channel.id}:{message.author.id}"
        plat_msg = PlatformMessage(
            platform="discord",
            session_id=session_id,
            user_id=str(message.author.id),
            text=message.content,
            reply_to_platform_id=str(message.channel.id),
        )
        reply = await agent_core.process(plat_msg)
        await message.channel.send(reply.text)
```

**踩坑点：**
- Discord 的 message 如果没有 `guild`（私信），需要特殊处理，`session_id` 生成逻辑要分支。
- 机器人需要 `Send Messages` 权限，且长文本需要自动分段（超过 2000 字符会自动报错）。可以封装一个 `send_long_message(channel, text, chunk_size=1900)` 工具。
- 如果 Agent 返回富文本或表格，推荐转换成 Discord embed，但为了跨平台一致，此处保持纯文本输出，复杂展示可交给未来优化。

### 4. Agent Core 统一处理

OpenClaw 的 Agent 通常是一个异步函数 `process(msg: PlatformMessage) -> AgentReply`。这里我们只展示一个极简框架：

```python
class AgentCore:
    async def process(self, msg: PlatformMessage) -> AgentReply:
        # 从共享 memory 中加载 session（Redis / 本地字典）
        # 调用 LLM 进行推理
        # 保存上下文
        return AgentReply(text="Echo: " + msg.text)
```

关键是 memory 的 session key 应使用 `session_id`，这样每个平台会话完全隔离。注意不要用全局变量保存上下文，否则 Telegram 用户和 Discord 用户的消息会互相污染。

### 5. 并发与部署

两个 Adapter 的 run 方法都在各自的 event loop 中，如果用 asyncio，可以把它们放在同一个 loop 里：

```python
async def main():
    tg_app = Application.builder().token(TG_TOKEN).build()
    tg_app.add_handler(MessageHandler(filters.TEXT, handle_telegram))
    discord_client = DiscordBot(intents=intents)

    await asyncio.gather(
        tg_app.run_polling(),
        discord_client.start(DISCORD_TOKEN),
    )
```

但 `tg_app.run_polling()` 本身内部有 `asyncio.run` 问题，需要改为手动 `tg_app.initialize()` + `tg_app.start()` + `updater = tg_app.updater; await updater.start_polling()` 以使用已有的 loop。同样 Discord 用 `start` 而不是 `run`。这步容易出错，网上搜到的很多教程是 `run_polling`，会导致无法并发。

## 可复用建议

- **抽象 Adapter 基类**，包含 `send_message`、`parse_message`、`handle_error` 接口。新增平台只需继承实现。
- **不要为每个平台单独维护 Prompt 或工具集**，所有平台共用一个 Agent 配置。如果有少量差异，可以在 `PlatformMessage` 中携带 `platform` 字段，让 Agent 在 Prompt 中感知而不改变核心逻辑。
- **做好速率限制**。Discord 对 API 调用有全局和分路由限制，Telegram 会报 429 错误。可以将 `send_message` 封装在自动重试/退避策略中。
- **日志要记录平台来源**，方便排查是哪个 Adapter 出了问题。
- **文件发送需统一抽象**，但目前阶段先保持纯文本输出，等文本跑稳再扩展。

## 总结

跨平台消息路由的本质不是给 Agent “接两个外设”，而是把平台当作一种 **IO 设备**，用适配器屏蔽差异。本例只用了几百行代码就使同一套 Agent 逻辑同时运行在 Telegram 和 Discord 上，后续扩展 Slack、WhatsApp 等也只需要新增适配器。

对于 OpenClaw 社区的实践者来说，这是让 Agent 从 demo 走向真正可用的关键一步。不要把时间花在与平台 API 的拉锯战上，将控制权还给 Agent 核心逻辑，才能把工程重点放在更有价值的任务编排和工具集成上。

---

