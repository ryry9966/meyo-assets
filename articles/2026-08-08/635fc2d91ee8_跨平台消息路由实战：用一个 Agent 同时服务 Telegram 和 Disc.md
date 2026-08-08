---
title: 跨平台消息路由实战：用一个 Agent 同时服务 Telegram 和 Discord
feedId: 32164
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

给自己的小项目做自动化时，经常遇到一个需求：希望同一个 Agent 既能回答 Telegram 群友的问题，又能响应 Discord 服务器的指令。虽然两个平台都提供了成熟的 Bot API，但各自的消息格式、会话模型、权限体系完全不同。如果为每个平台写一套重复的逻辑，Agent 的核心能力很容易被适配代码吃掉，维护成本也成倍增加。

过去一年我在 OpenClaw 生态里积累了一套比较务实的方案：用统一消息模型 + MCP（Model Context Protocol）工具层实现消息路由，让 Agent 只关心“收到什么、该回什么”，至于通过哪个平台发回去，由路由层自动解决。这篇文章还原一下设计思路、关键步骤和实际踩过的坑。

## 问题拆解

把问题分解一下，跨平台消息路由需要解决三件事：

1. **入站统一**：把 Telegram 的 `Message`、Discord 的 `Message` 都转成同一种内部结构，保留平台和会话特征。
2. **出站路由**：Agent 回复时只调用一个抽象的“发送消息”工具，这个工具根据会话元数据自动选择正确的平台和频道。
3. **会话防污染**：同一个 Agent 可能同时处理多个平台的多个对话，必须保证每个会话的上下文隔离，且不会把 A 平台的内容误发到 B 平台。

下面的方案建立在 **Agent + MCP 工具** 的架构上，你可以按自己的技术栈替换底层通信方式（HTTP、WebSocket、消息队列都可以），但核心思想不变。

## 做法与步骤

### 1. 定义统一消息结构

不要在各平台的原生对象上做业务逻辑。我定义了一个轻量的内部消息体：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" or "discord"
    session_id: str        # 组合键，如 "telegram:123456"
    user_id: str
    text: str | None
    attachments: list[str] # 附件本地路径或URL
    timestamp: float
```

`session_id` 是关键设计：它把平台标识和平台内的会话 ID 拼在一起，保证全局唯一。之后 Agent 的所有上下文、记忆、工具调用都围绕这个 `session_id` 展开。

### 2. 编写平台适配器

适配器只做两件事：接收平台事件并转换成 `UnifiedMessage`，以及根据 `session_id` 把文本发回对应平台。

以 Telegram 为例，用 `python-telegram-bot` 的 `Application` 监听消息：

```python
async def tele_handler(update, context):
    chat_id = update.effective_chat.id
    um = UnifiedMessage(
        platform="telegram",
        session_id=f"telegram:{chat_id}",
        user_id=str(update.effective_user.id),
        text=update.message.text,
        ...
    )
    await agent_queue.put(um)
```

Discord 侧用 `discord.py` 的 `on_message`：

```python
async def on_message(message):
    if message.author.bot: return
    um = UnifiedMessage(
        platform="discord",
        session_id=f"discord:{message.channel.id}",
        user_id=str(message.author.id),
        text=message.content,
        ...
    )
    await agent_queue.put(um)
```

两个适配器各自把消息扔进同一个 `asyncio.Queue`，Agent 主循环统一消费这个队列即可。

### 3. 通过 MCP 工具统一出站

Agent 需要回复时，不应直接依赖 `telegram.send_message` 或 `discord.Messageable.send`。我把它包装成一个 MCP 工具 `send_message_to_platform`：

```json
{
  "name": "send_message_to_platform",
  "description": "向当前会话发送文本消息。平台和会话由上下文自动确定。",
  "parameters": {
    "text": {"type": "string", "description": "要发送的文本内容"}
  }
}
```

MCP server 实现时，会从当前请求上下文里取出 `session_id`，解析出 `platform` 和平台侧的会话 ID，然后调用对应的适配器函数：

```python
if platform == "telegram":
    await tele_bot.send_message(chat_id=chat_id, text=text)
elif platform == "discord":
    channel = discord_client.get_channel(int(chat_id))
    await channel.send(text)
```

在 OpenClaw 或任何支持 MCP 的 Agent 框架里，Agent 只需要调用这个工具，完全不用感知底层是哪一家。

### 4. 会话与上下文维护

Agent 的上下文通常需要按 `session_id` 隔离。我在 Redis 里为每个 `session_id` 维护一个消息列表，每次进入主循环时取出历史消息拼成 prompt，处理完后再把 assistant 的回复追加回去。

这里有个常被忽视的点：**Agent 的并发处理必须针对同一个 session 串行化**。如果同一个 Telegram 群同时有两条消息进来，不可以开两个协程同时写 session 的上下文，否则历史消息会乱序。简单的做法是为每个 `session_id` 分配一个 `asyncio.Lock`，在取出消息前加锁，处理完解锁。

## 踩坑点

- **富文本与附件格式差异**  
  Discord 的嵌入块、按钮、下拉菜单，在 Telegram 上没有完全等价的表达。目前方案只保证纯文本通路可靠，富元素统一降级为文本提示；附件转发需要额外处理 CDN 鉴权，跨平台发送附件建议先上传到公共存储再发链接。

- **会话 ID 的精细度**  
  Telegram 私聊的 `chat_id` 和群聊的 `chat_id` 可以直接用作会话，但 Discord 中同一个 channel 的不同 thread，`channel.id` 是一样的，需要额外加上 `thread.id`。忘记这一点会导致不同讨论串的消息混进同一个 session。

- **速率限制与重试**  
  两个平台都有明确的 API 限频，Agent 连续回复多条消息时极容易触发。我在出站适配器里加入了令牌桶和指数退避重试，同时 Agent 侧被设计为 “尽量一条回复解决问题”，避免连续吐出五六条短句。

- **上下文泄漏**  
  Agent 的 prompt 里绝对不能出现其他平台的 `session_id` 或敏感内容。所有上下文存储的 key 都用平台+会话 ID 隔离，并且在 MCP 工具调用时，平台适配器只允许向当前 session 绑定的平台发送消息。

## 可复用建议

- **适配器接口标准化**：抽象一个 `BaseAdapter`，定义 `inbound() -> UnifiedMessage` 和 `outbound(session_id, text)`，新平台接入只需实现这两个方法。
- **MCP 工具按平台主动注册**：如果你的场景需要不同平台拥有不同的工具集，可以在 MCP server 里根据 session 的平台动态暴露工具列表，但绝大多数时候一个 `send_message_to_platform` 足够了。
- **监控与降级**：为每个平台适配器增加心跳和错误计数，当某个平台连续失败时自动熔断，避免拖垮整个 Agent 进程。
- **Docker 化部署**：把 Agent 主进程、各平台适配器、Redis 放在同一个 docker-compose 里，适配器通过 HTTP 回调或共享 asyncio loop 通信均可，方便本地复现。

## 总结

跨平台消息路由的技术核心不在于多复杂的框架，而在于**统一消息模型 + 会话ID平台化绑定 + MCP 工具抽象**这三板斧。用 MCP 把出站能力封装成工具后，Agent 本身的逻辑变得非常干净，后续增加 WhatsApp、Line 等新平台也只需要实现一套适配器接口，几乎不影响 Agent 代码。

这个方案在 OpenClaw 的插件生态里跑了一段时间，稳定性足够支撑小团队使用。如果你也在折腾类似的自动化，建议从最简单的纯文本路由做起，先把会话隔离和速率限制处理好，再逐步扩展附件的跨平台转发。

---

