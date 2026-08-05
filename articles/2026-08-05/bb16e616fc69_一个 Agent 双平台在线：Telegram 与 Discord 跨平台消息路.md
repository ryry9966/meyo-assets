---
title: 一个 Agent 双平台在线：Telegram 与 Discord 跨平台消息路由实践
feedId: 31763
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

在用 OpenClaw 构建对话 Agent 时，最常见的场景是让 Agent 接入某个即时通讯平台（如 Telegram 或 Discord）。但如果用户群体分布在多个平台，为每个平台单独维护一个 Agent 实例不仅浪费资源，还会导致行为不一致、记忆割裂。于是自然诉求变成：**同一个 Agent，能否同时服务 Telegram 和 Discord 两个社群，并且共享上下文与能力？**

这篇文章记录了一次基于 OpenClaw 的工程实践：在不侵入 Agent 核心逻辑的前提下，通过一个轻量级消息路由层，让一个 Agent 实例同时处理来自 Telegram 和 Discord 的用户消息。

目标受众是熟悉 OpenClaw、MCP、插件机制，并且自己搭建过 Bot 的自动化实践者。不做产品推广，只讲做法和坑。

---

## 问题拆解

让 Agent 服务两个平台，本质上要解决三个问题：

1. **入站消息归一化**  
   Telegram 用 `Message` 对象，Discord 用 `Interaction` 或 `Message`，字段结构、媒体处理、回复引用方式都不一样。需要抽象出一个统一的内部消息格式，屏蔽平台差异。

2. **Agent 回复分派**  
   Agent 返回的结果可能是文本、卡片、Markdown 甚至图片，必须依据消息来源平台，把回复转换成平台支持的格式，再调用对应的 API 发出。

3. **会话与身份映射**  
   两个平台的用户 ID、群组 ID 完全独立，不能简单用 `user_id` 当会话键。需要组合 `platform + chat_id` 建立会话标识，并保证长期记忆（如 OpenClaw 的 session）能正确关联。

---

## 做法与步骤

### 1. 定义统一消息模型

不直接依赖平台 SDK 的数据结构，自定义内部 `IncomingMessage`：

```python
@dataclass
class IncomingMessage:
    platform: str          # "telegram" | "discord"
    chat_id: str           # 组合键：platform + 平台 chat/guild id
    user_id: str           # 平台用户 ID
    text: str | None
    attachments: list[Attachment]
    reply_to: str | None   # 引用的消息 ID（如有）
```

Attachments 也统一为 `(url, mime_type)` 的列表。这样 Agent 只需要面对一种消息模型。

### 2. 编写平台 Adapter

为每个平台实现两个方法：`parse_input` 与 `send_reply`。

**Telegram Adapter**  
- 使用 `python-telegram-bot` 或直接 `aiohttp` 消费 webhook  
- `parse_input` 从 `Update` 对象提取 `chat_id`、`user_id`、`text` 等  
- `send_reply` 根据 Agent 输出调用 `sendMessage`，注意 Markdown 转义和 4096 字符截断策略

**Discord Adapter**  
- 使用 `discord.py` 的 `on_message` 事件，过滤 Bot 自身消息  
- `parse_input` 将 `discord.Message` 映射到 `IncomingMessage`  
- `send_reply` 使用 `message.reply()` 或 `channel.send()`，长文本拆段发送，必要时使用 `embed` 发送代码块（避免 Discord 格式化限制）

### 3. 消息路由器

路由器的职责：接收 Adapter 传入的 `IncomingMessage`，生成会话键 `session_id = f"{platform}:{chat_id}"`，然后将消息交给 OpenClaw 的 Agent。

伪代码结构如下：

```python
async def handle_message(msg: IncomingMessage):
    session_id = f"{msg.platform}:{msg.chat_id}"
    agent = get_agent(session_id)          # 获取或创建 session 级 Agent
    reply = await agent.run(msg.text, context=msg)
    await dispatch_reply(msg.platform, msg.chat_id, reply, msg)
```

`dispatch_reply` 根据 `platform` 调用对应的 Adapter 发送回复，同时处理附件、卡片、按钮等结构化回复。

### 4. Agent 与会话管理

这里利用 OpenClaw 的会话隔离能力：每次调用 `get_agent(session_id)` 时，若不存在则初始化一个新的 Agent 上下文（加载 system prompt、工具列表、记忆）。由于 `session_id` 已包含平台前缀，Telegram 群组的对话和 Discord 频道的对话互不干扰，但可以被同一个 Agent 配置文件管理，共享相同的系统指令与 MCP 工具。

### 5. 配置与部署

用 `supervisor` 或 `systemd` 跑一个 Python 进程，同时启动：

- Telegram webhook（用 `aiohttp` 在同一进程监听，注意 webhook 需要公网 HTTPS，用 nginx 反代即可）
- Discord bot 的 `asyncio` 事件循环（通过 `discord.py` 的 `start()` 与 Telegram webhook 共用一个 event loop）

关键点：**保证两个 Adapter 在同一个 asyncio event loop 下运行**，否则 Agent 实例共享会出现并发问题。我们在实践中使用了 `asyncio.gather` 同时运行两个入口，彻底规避多进程同步烦恼。

---

## 踩坑记录

- **速率限制 429**  
  Discord 对同一频道每秒发送消息有限制，Telegram 也有类似限制。Agent 如果短时间内回复大量消息（例如批量处理），极易触发 429。解决方法：实现一个基于令牌桶的异步发送队列，超出速率时自动延迟。

- **消息格式差异**  
  Telegram 支持 `MarkdownV2`、`HTML`，Discord 用 Markdown 但子集不同。Agent 输出的 Markdown 不能直接套用。实践中我们让 Agent 输出纯净文本，由 Adapter 负责转义。如果 Agent 需要生成代码块，我们会将代码块转换为 Discord `embed`，避免格式丢失。

- **长消息分段**  
  Telegram 单条消息限制 4096 字符，Discord 为 2000 字符。必须实现分段器，且尽量在句子边界切割，避免分割代码块。我们采用 TextSplitter，监控未闭合的代码块并在下一段自动补上前后标记。

- **回复目标错位**  
  Discord 频道内多条消息并发时，简单调用 `channel.send()` 会丢失回复关系。务必利用 `message.reply()` 或 `message.reference` 明确被回复消息。Telegram 侧需要带上 `reply_to_message_id`。

- **Webhook 接收去重**  
  Telegram 在某些情况下可能重复发送同一更新，需要在 Adapter 中记录最近的 `update_id` 并去重，避免 Agent 重复处理。

---

## 可复用建议

1. **平台抽象接口**：定义 `IPlatformAdapter` 包含 `parse_input` 和 `send_reply` 方法，方便未来接入 WhatsApp、Slack 等。
2. **消息队列解耦**：若 Agent 处理逻辑较重，可以在路由器前加一个 `asyncio.Queue`，让 Adapter 生产消息，Worker 消费，避免阻塞 webhook。
3. **配置集中管理**：所有平台 token、速率限制参数放在一个 `config.yaml` 中，方便切换环境。
4. **可观测性**：为每个消息生成一个 `trace_id`，记录从接收到回复的全过程日志，跨平台排查问题时有奇效。
5. **Agent 能力对齐**：如果不同平台需要不同的系统提示（比如 Discord 社群偏技术、Telegram 偏闲聊），可以通过 `platform` 参数动态注入前缀，无需维护多个 Agent 实例。

---

## 总结

一个 Agent 服务多平台的核心不是“把两个 Bot 写在一起”，而是**用工程化手法将平台差异封装在 Adapter 层，让 Agent 只面对统一的消息模型和会话标识**。OpenClaw 的 session 机制天然适合这个模式，配合少量自建路由代码，就能把 Agent 的能力同时投射到 Telegram 和 Discord 里。

这套架构已在内部稳定运行数周，日处理消息量约 2000+，没有出现串话或发错平台的情况。扩展性也足够：接入新平台只需实现对应的 Adapter，Agent 部分无需任何改动。

如果你也在维护跨平台 Bot，不妨试试这套思路——让 Agent 专注对话，把路由交给 Adapter。

---

