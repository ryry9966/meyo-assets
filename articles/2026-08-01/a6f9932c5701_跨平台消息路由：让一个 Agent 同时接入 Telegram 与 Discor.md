---
title: 跨平台消息路由：让一个 Agent 同时接入 Telegram 与 Discord
feedId: 31149
source: 综合讨论
publishedAt: 2026-08-01
---

在用 Agent 框架（如 OpenClaw）构建自动化服务时，我们经常需要让同一个业务逻辑同时服务多个即时通讯平台——最常见的就是 Telegram 和 Discord。与其为每个平台单独维护一套 Agent 实例，不如设计一套统一的跨平台消息路由，让一个 Agent 处理好所有渠道的消息。

## 背景

OpenClaw 这类框架已经通过 MCP 或插件体系提供了灵活的扩展能力，使得我们可以在不修改核心逻辑的情况下接入不同的平台。当需要同时支撑 Telegram Bot 和 Discord Bot 时，挑战不在于“能不能连上”，而在于如何保持会话上下文、指令分发、错误处理和回复格式的一致性，同时不让适配代码污染核心业务。

## 问题拆解

本质上，我们要解决两个主要问题：

1. **入站归一化**：将 Telegram 和 Discord 各自的消息格式、元数据字段转化为统一的内部事件。
2. **出站适配**：根据消息来源，将 Agent 的通用回复以各自平台所需的形式发送回去（包括长文本拆分、Markdown 语法的差异化处理、附件引用等）。

此外还需要考虑：会话状态跨平台隔离还是共享、命令冲突处理、异步任务调度等。

## 做法 / 步骤

在一个实际工程实现中，我采用了以下分层结构：

```
Telegram Adapter ─┐
                   ├─ Unify Event Bus ─ Agent Core ─ Reply Router ──┬─ Telegram Sender
Discord Adapter  ─┘                                                 └─ Discord Sender
```

### 1. 定义统一消息模型

所有平台消息先被转换为一个内部 dataclass，比如：

```python
@dataclass
class UnifiedMessage:
    platform: str                # "telegram" / "discord"
    user_id: str
    chat_id: str                 # TG 的 chat_id，DC 则组合 guild+channel
    guild_id: Optional[str]
    text: str
    attachments: List[dict]      # 统一附件描述
    raw: dict                    # 原始事件，以备不时之需
```

这样核心处理逻辑不需要关心平台细节。

### 2. 实现适配器

基于 OpenClaw 的 MCP 机制，分别实现 `TelegramAdapter` 和 `DiscordAdapter`，它们负责：

- 连接各自的 Bot API / Gateway
- 接收消息，转换成 `UnifiedMessage` 并推入统一事件队列
- 暴露 `send_message` 方法，根据平台特性发送回复

关键点是保持适配器无状态（或只维护连接），所有业务状态（如对话历史）由 Agent 内核通过 Redis 等外部存储管理。

### 3. 会话上下文处理

会话 ID 的设计需要能区分不同平台的同一用户。我采用了复合键：

```
platform:user_id:chat_id
```

对于 Discord，`chat_id` 实际是 `channel_id`，以免 DM 和服务器频道的同一用户混淆。TG 则直接用 `chat_id`。所有对话记忆、用户偏好都绑定在这个会话键上，通过 OpenClaw 的上下文管理插件进行存取。

### 4. 指令路由与平台特化

指令解析放在统一入口，但某些命令可能只在特定平台有意义（如 `/serverinfo` 只对 Discord 有效）。做法是在命令注册时增加 `platforms` 字段，不匹配的平台直接返回友好提示。更细粒度的差异通过平台感知的回复模板处理。

### 5. 长文本拆分与格式适配

Telegram 消息上限 4096 字符，Discord 则是 2000。Agent 回复如包含代码块、列表等 Markdown，直接截断会破坏格式。我写了一个基于 Markdown AST 的拆分器，优先在段落边界断开，保证每个片段格式合法。同时，Discord 对 Markdown 的支持比 Telegram 有限（不支持 HTML，代码块语言高亮有差异），需要在发送前做一次格式降级。

### 6. 错误处理与速率限制

两个平台 API 都有速率限制，但策略不同。适配器内集成各自的退避重试逻辑：Telegram 使用官方建议的 429 重试，Discord 则根据 `X-RateLimit-*` 头动态调整。关键是要避免一个平台的 429 阻塞另一个平台的消息发送，因此发送调用必须异步且互相隔离。

## 踩坑点

- **消息附件跨平台引用**：Telegram 图片/文件直接提供 `file_id`，而 Discord 是 CDN 链接，如果 Agent 需要在回复中引用这些附件，必须在统一消息模型里封装为平台无关的 URL 或内部存储引用，不能直接透传平台 ID。
- **异步事件循环冲突**：两个适配器通常会各自运行一个事件循环（如 `python-telegram-bot` 和 `discord.py`），容易造成 asyncio 冲突。推荐的方法是将两个适配器作为独立子进程，通过消息队列（Redis pub/sub 或 NATS）与 Agent 核心通信，彻底解耦。
- **会话状态泄漏**：如果复合键设计不当，TG 的群聊消息可能被误认为同一个会话，导致上下文混乱。务必确认 `chat_id` 的粒度与业务场景匹配。
- **命令冲突**：Telegram 的 `/start` 和 Discord 的斜杠命令体系不完全兼容。我们为 Discord 单独注册了应用命令（Application Command），并在适配器中将这些交互转换为统一命令事件。

## 可复用建议

- **抽象统一的 Adapter 接口**，包含 `listen()` 和 `send(UnifiedReply)`，未来添加新平台（如 Slack、WhatsApp）只需实现该接口。
- **使用中间件链**处理消息预处理：例如基于 OpenClaw 的中间件机制进行用户鉴权、敏感词过滤、频率限制，这些与平台无关。
- **集中监控**：将每个平台的消息量、延迟、错误率统一输出到 metric 系统，以便快速定位是某个渠道的 API 异常还是 Agent 本身性能问题。
- **配置驱动**：平台令牌、速率限制参数、命令白名单等均放在配置文件中，便于动态切换测试环境。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，在工程上完全可行，核心在于建立一套平台无关的消息模型，并坚持适配器与业务逻辑的彻底解耦。这样不仅减少了重复代码，也使团队可以快速响应新增平台的诉求。对于 OpenClaw 用户，充分利用其 MCP 插件体系能够让这种架构落地得更加干净，避免陷入特定 SDK 的细节泥潭。

---

