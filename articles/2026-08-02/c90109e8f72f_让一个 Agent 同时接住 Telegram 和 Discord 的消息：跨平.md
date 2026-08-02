---
title: 让一个 Agent 同时接住 Telegram 和 Discord 的消息：跨平台路由实战
feedId: 31347
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

很多自建 Agent 项目一开始只接一个平台，比如 Telegram。一旦社区壮大，Discord 上也有群组需要同样的自动化能力，常见的做法是另外维护一套 Discord bot，逻辑复制一份。很快你会发现两个 bot 开始“分叉”——修复一个平台的 bug 忘记同步到另一个，功能实现出现差异，用户在不同平台得到不一致的体验。

更理想的方式是：让**同一个 Agent 实例**同时服务 Telegram 和 Discord，统一处理逻辑，只在最后回复时适配各自平台的消息格式。借助 OpenClaw 的插件体系和 MCP 工具调用，这完全可行。本文记录一次从单平台扩展到双平台的跨平台消息路由实践，涵盖架构、实现要点和踩坑记录。

## 问题拆解

要让一个 Agent 处理两条通道，需要解决几个关键问题：

1. **消息格式差异**：Telegram 支持 MarkdownV2/HTML，Discord 使用 Markdown 子集，且倾向于“软换行不自动合并”。两者的消息分片长度限制也不同（TG 4096，Discord 2000）。
2. **入口归一化**：无论是 TG webhook 还是 Discord gateway 事件，都要转换成相同的内部消息结构，并携带“平台来源”和“回复回调”。
3. **身份与会话**：同一个自然人在 TG 和 Discord 上的 ID 完全不同，如果需要跨平台延续对话上下文，需要额外绑定。本文场景是每个平台独立会话，但共享 Agent 的推理能力与工具库。
4. **并发与资源隔离**：两个平台的网络模型不同（长轮询/Webhook vs 持久 Gateway WebSocket），不能互相阻塞。Agent 的状态管理要避免跨会话污染。

## 做法与步骤

### 1. 定义统一消息模型

```python
from dataclasses import dataclass
from typing import Optional, Callable, Awaitable

@dataclass
class UnifiedMessage:
    platform: str          # "telegram" | "discord"
    chat_id: str           # 唯一标识一个对话上下文
    user_id: str           # 发送者 ID
    text: str
    reply_to_message_id: Optional[str] = None
    # 回复回调，Agent 处理结束后调用
    reply_callback: Optional[Callable[[str], Awaitable[None]]] = None
```

`chat_id` 在 TG 就是 chat id（字符串形式），在 Discord 可以使用 `channel_id`，如果需要线程上下文则拼接 `channel_id:thread_id`。

### 2. 实现平台适配器

**Telegram 端（python-telegram-bot + webhook）**

```python
from telegram.ext import Application, MessageHandler, filters
from telegram.request import HTTPXRequest

async def tg_handler(update, context):
    msg = UnifiedMessage(
        platform="telegram",
        chat_id=str(update.effective_chat.id),
        user_id=str(update.effective_user.id),
        text=update.message.text or "",
        reply_callback=lambda text: update.message.reply_text(text, parse_mode="MarkdownV2")
    )
    await agent.process(msg)
```

踩坑：`parse_mode="MarkdownV2"` 时，`-`、`.`、`!` 等字符需要转义，否则 TG 返回 400 错误。建议在 reply_callback 内部做逃逸处理，或者统一发送纯文本，Markdown 留给富内容场景单独适配。

**Discord 端（discord.py）**

```python
@bot.event
async def on_message(message):
    if message.author.bot:
        return
    msg = UnifiedMessage(
        platform="discord",
        chat_id=str(message.channel.id),
        user_id=str(message.author.id),
        text=message.content,
        reply_callback=lambda text: message.channel.send(text[:2000])
    )
    await agent.process(msg)
```

Discord 消息 2000 字符限制，这里简单切片，后面提到更稳健的方式。

### 3. 消息路由器与 Agent 接入

路由器本身只是把统一消息塞给 Agent 核心。核心做意图识别、调用 MCP 工具、生成回复，最后调用 `reply_callback` 发送出去。如果 Agent 是有状态的，可以按 `(platform, chat_id)` 隔离 session：

```python
sessions: dict[str, AgentSession] = {}

async def process(msg: UnifiedMessage):
    session_key = f"{msg.platform}:{msg.chat_id}"
    session = sessions.get(session_key)
    if not session:
        session = AgentSession()
        sessions[session_key] = session
    reply_text = await session.run(msg.text, tools=available_tools)
    await msg.reply_callback(reply_text)
```

> 注意：如果使用 OpenClaw 的内置 Agent 运行器，它已经提供了 session 管理，可将平台的 `chat_id` 映射为 OpenClaw 的 `session_id`，直接利用其持久化与限流机制。

### 4. 解决两个事件循环的共存问题

TG 的 `Application.run_webhook` 和 Discord 的 `bot.run()` 都想要主导事件循环。如果单纯各运行一个，可能冲突。推荐使用同一个 `asyncio` 事件循环，手动管理：

```python
async def main():
    # 初始化 TG webhook 应用
    tg_app = Application.builder().token(TG_TOKEN).build()
    tg_app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, tg_handler))
    await tg_app.initialize()
    await tg_app.start()
    # 初始化 Discord
    async with discord_client:
        discord_client.loop.create_task(tg_app.updater.start_webhook(...))
        await discord_client.start(DISCORD_TOKEN)
```

或者更简单：TG 端使用 `webhook` 模式 + FastAPI，Discord 用单独线程起一个 `discord.Client`，将消息推入同一个 `asyncio.Queue`，由 Agent 协程消费。

## 踩坑记录

### Telegram 转义与超长消息

- 如果 Agent 输出含 Markdown 特殊字符，而 `parse_mode="MarkdownV2"`，必须用 `escape_markdown()` 处理用户输入部分，保留 Agent 格式部分。建议构建回复时指定格式版本，或统一用 HTML 模式（转义规则不同）。
- 消息超过 4096 字符时，TG API 直接拒绝。需要实现分段发送逻辑。注意分段时不要破坏 Markdown 结构（如粗体跨片）。

### Discord 的分段与 Embed 陷阱

- `message.channel.send()` 超过 2000 字符同样报错。可封装一个 `send_long_message(channel, text)`，按换行符或空格断句，保证每段合法。
- 不要滥用 Embed：很多频道关闭 Embed 权限，太长 Markdown 在手机端显示差。默认使用纯文本+代码块是比较稳妥的选择。

### 会话状态膨胀

- 如果 Agent 为每个对话保持上下文（历史消息），长期运行会导致内存膨胀。需要设置会话 TTL，过期自动清除。
- 使用 OpenClaw 的 session 存储后端（如 Redis）来实现持久化，并配置最大 token 窗口，防止上下文无限增长。

### 身份绑定

- 如果希望同一用户在 TG 和 Discord 之间延续对话，必须身份绑定。简单实现可定义一个命令 `!bind` 生成一次性令牌，用户在另一平台发送该令牌完成关联。但会引入复杂的状态机，需评估是否真的需要。

## 可复用建议

- **抽象平台适配接口**：定义 `PlatformAdapter` 抽象类，实现 `parse_incoming` 和 `send_reply`，未来添加 WhatsApp、Slack 只需写新适配器。
- **统一错误处理**：各平台的 `reply_callback` 中捕获异常（如 network blip、权限不足），记录日志并触发告警，不要中断 Agent 流水线。
- **配置外挂**：将所有 token、webhook URL、速率限制参数放在环境变量或配置文件，方便多环境部署。
- **利用 MCP 工具**：Agent 调用的外部工具（如查询知识库、调用 API）通过 MCP server 提供，完全与平台解耦，保证所有平台收到相同质量的结果。
- **监控与可观测性**：记录每个平台的消息量、延迟、错误率。可暴露 Prometheus 指标，追踪 `platform` 标签。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，本质是**在入口层做归一化，在出口层做适配**。核心逻辑完全复用，不仅降低维护成本，还保证了用户体验的一致性。实现过程中需要处理好格式差异、事件循环共存和会话生命周期管理等工程细节。配合 OpenClaw 的 session 管理和 MCP 工具扩展，这种跨平台架构可以快速推广到更多通道，为 Agent 赋予真正的“多租客”服务能力。

---

