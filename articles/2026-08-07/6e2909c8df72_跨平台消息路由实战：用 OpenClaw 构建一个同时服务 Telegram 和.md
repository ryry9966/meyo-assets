---
title: 跨平台消息路由实战：用 OpenClaw 构建一个同时服务 Telegram 和 Discord 的 Agent
feedId: 31947
source: 综合讨论
publishedAt: 2026-08-07
---

## 为什么需要跨平台路由

社区里跑单平台 Agent 的做法已经很成熟了：开一个 Telegram bot，接上 OpenClaw 的 `telegram` 通道，再加个简单的 prompt，就能让 LLM 在群里回消息。但实际场景中，你的用户可能一部分在 Telegram，一部分在 Discord，甚至同一个团队用两种工具。每新增一个平台就维护一套独立实例，配置散落，会话状态也无法共享，长期看是负债。

更合理的方式是让一个 Agent 实例同时监听、处理并回复多个平台的消息。这样 rule、工具、记忆都在一个执行上下文中，维护成本低，体验也能保持一致。这篇文章记录我在 OpenClaw 上实现 Telegram + Discord 双通道消息路由的过程，重点放在工程细节和踩坑点上，希望对你复用有帮助。

## 问题拆解

要同时接入两个 IM 平台，核心难点不是“能不能收到消息”，而是三个工程化问题：

1. **消息格式异构** – Telegram 用 `Message` 对象（包含 `chat.id`、`from.id`、`text`、`photo` 等），Discord 用 `MessageCreate` 事件（包含 `guild_id`、`channel_id`、`author.id`、`content`、`attachments`）。字段名、附件处理方式、用户标识都不同。
2. **连接模式不同** – Telegram 推荐 webhook，Discord 一般用 Gateway WebSocket。如果要跑在同一个进程里，需要一个能同时管理两种长连接/回调的 runtime。
3. **会话隔离** – 同一个用户在两个平台上可能是完全不同的 ID，如何在不耦合平台信息的前提下维持对话上下文？

解决思路是在 OpenClaw 的 channel 抽象之上，搭建一个薄薄的 **消息归一化层** 和 **路由回复层**，Agent 本体只依赖统一的消息接口。

## 做法与步骤

### 1. 项目骨架

我用的 OpenClaw 版本支持通过配置文件同时启动多个 channel。新建一个 OpenClaw 项目，目录结构大致如下：

```
project/
  openclaw.config.yaml
  agent.py
  channels/
    telegram_adapter.py
    discord_adapter.py
  router.py
```

`openclaw.config.yaml` 里声明两个 channel 的入口：

```yaml
channels:
  telegram:
    type: telegram
    token: ${TG_BOT_TOKEN}
    webhook_path: /tg
  discord:
    type: discord
    token: ${DISCORD_BOT_TOKEN}
```

OpenClaw 在启动时会根据配置自动拉起 Telethon/HTTP 服务器和 Discord 的 Gateway 连接，并把两个通道的入站消息都推到一个统一的 event bus 里。

### 2. 统一消息模型

最关键的一步是定义内部消息结构，屏蔽平台差异。我设计了一个 `UnifiedMessage`，字段足够覆盖常用场景：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" 或 "discord"
    user_id: str           # 全局唯一用户标识（见下文）
    chat_id: str           # 会话/群组标识
    text: Optional[str]
    attachments: List[Attachment]
    raw_platform_event: Any  # 保留原始对象，方便特殊逻辑
```

用户标识生成规则：`{platform}:{平台用户ID}`。这样在 LLM 的记忆里用户不会串，事后也可以很方便地做跨平台关联（比如绑定数据库）。

### 3. 适配器：平台消息 -> UnifiedMessage

分别在 `telegram_adapter.py` 和 `discord_adapter.py` 中实现转换函数。以 Telegram 为例：

```python
def tg_to_unified(update: Update) -> UnifiedMessage:
    return UnifiedMessage(
        platform="telegram",
        user_id=f"tg:{update.effective_user.id}",
        chat_id=str(update.effective_chat.id),
        text=update.message.text,
        attachments=parse_tg_attachments(update.message),
        raw_platform_event=update
    )
```

Discord 适配器同理，注意附件需要从 `message.attachments` 转换为一致的 `Attachment`（包含 url、type）。两个适配器都注册到 event bus，OpenClaw 每收到一条平台消息就自动走对应的转换，产出 `UnifiedMessage`。

### 4. Agent 处理与回复路由

Agent 部分就是一个标准的 OpenClaw agent 定义，只不过它只处理 `UnifiedMessage`，并返回 `UnifiedReply`（包含 `target_platform`、`chat_id`、`text` 等）。`router.py` 根据 `target_platform` 把回复推回正确的 channel adapter，adapter 再调用平台 API 发消息。

会话隔离使用 `(platform, chat_id)` 作为 session key，OpenClaw 的 memory 后端用 Redis 时天然支持这种复合 key 的隔离，不用改代码。

### 5. 运行与监控

启动后两个 bot 会同时在线。在 Telegram 里发 `/start`，Discord 里发 `!hello`，Agent 都能正常响应。为了避免某个平台掉线影响另一个，我加了心跳检测：每 30 秒检查一次 discord gateway 和 telegram webhook 状态，异常时单独重连，不重启整个进程。

## 踩坑记录

**1. Webhook + WebSocket 同进程的资源竞争**

Telegram webhook 通常用 aiohttp 或 FastAPI 起 HTTP server，而 Discord 需要维持一个持久 WebSocket。如果 OpenClaw 内置的运行时没有处理好事件循环，两者会互相阻塞。解决办法是让 WebSocket 和 HTTP server 跑在同一个 event loop，但使用不同 port 或子路径；对于 OpenClaw，直接使用它推荐的 `openclaw serve` 命令即可，它内部已用 asyncio.gather 处理了。

**2. 附件下载与重传的坑**

Telegram 的文件需要用 `bot.get_file()` 再通过 `file_path` 下载，而 Discord 的附件直接提供临时 URL。如果 Agent 需要将一张图片从 TG 转发到 Discord，必须先把文件下载到本地（或流式传到 Discord API），因为 TG file_path 是内部链接，外部无法访问。这在做跨平台通知时很容易踩。建议在适配器中统一做附件处理：收到附件时预下载到临时存储，传递给 Agent 的 `Attachment` 对象里带上本地路径或可公网访问的 URL。

**3. 速率限制和重试**

Telegram 的限制相对透明，Discord 的全局 rate limit 一旦触发会整个 bot 被禁言几秒。我在 router 里加了一个简陋的令牌桶，对每个平台单独限流，超出请求直接排队（或丢弃，取决于配置）。这比让平台侧直接返回 429 后 Blind Retry 更可控。

**4. 命令前缀冲突**

TG 用 `/`，Discord 用 `!`，如果 Agent 本身依赖前缀做指令分发，需要在 UnifiedMessage 中提前识别出“意图”，而不是在平台层硬编码。我把指令识别逻辑放在 Agent 的 prompt 阶段，让 LLM 自己理解，避免了平台差异。

## 可复用建议

- **抽象先行**：哪怕一开始只接一个平台，也建议用 `UnifiedMessage` 的形式写 Agent，后续加平台只是加一个适配器文件。
- **会话 key 设计**：平台前缀 + 原生 ID，简单有效，之后可以引入用户绑定表做跨平台合并。
- **不要完全隐藏平台信息**：`raw_platform_event` 保留原始事件，方便在 Agent 中实现“仅限 Telegram 的特定操作”这类特化需求。
- **将重连逻辑内聚在 channel 层**：不要试图在 Agent 里处理连接问题，保持关注点分离。

## 总结

用 OpenClaw 做跨平台消息路由，本质上就是定义一个统一的消息壳子，然后让 Agent 只关心业务逻辑，平台差异由适配器消化。Telegram 和 Discord 双通道实现下来，核心代码（适配器 + router）不到 300 行，维护负担比两个独立实例小得多。下一步可以考虑加入 WhatsApp 或飞书，架构已经预留好了扩展空间。

如果你的 Agent 也要开始跨平台服务用户，这套模式值得一试。

---

