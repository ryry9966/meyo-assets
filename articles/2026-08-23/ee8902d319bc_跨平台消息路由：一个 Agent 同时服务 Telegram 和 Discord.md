---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34348
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 这类 Agent 运行时上，我们常把同一个助手接到不同聊天平台。Telegram 和 Discord 是最常见的两个，但两边 API 模型差异很大：Telegram 主要是长轮询 getUpdates 或 webhook，Discord 则依赖 Gateway/WebSocket 或 HTTP webhook。若直接在一个 Agent 核心里写两个平台的收发逻辑，很快会陷入重复代码和状态混乱。

本文记录一次将单个 Agent 同时接入 Telegram 和 Discord 的工程实践，重点不是平台 API 教程，而是中间的消息路由层如何设计，才能让 Agent 核心保持无平台状态。

## 问题

一开始最容易犯的错，是让 Agent 直接调用 python-telegram-bot 和 discord.py 的 API，业务逻辑里出现大量 `if platform == 'tg'`。主要痛点：

- 消息结构不同：Telegram 的 `message_id`、`chat_id`、`from` 与 Discord 的 `message.id`、`channel_id`、`author` 命名和类型都不一样。
- 回复方式不同：Telegram 可直接 `reply_to_message_id`，Discord 则需要 `message_reference`。
- 长度限制不同：Telegram 文本上限 4096，Discord 普通消息 2000。
- 身份体系不同：Telegram 用户是数字 ID，Discord 是 snowflake 字符串，需要统一主体标识。
- 限流与断线策略不同：Telegram 有 flood control，Discord Gateway 有 identify 频率限制。

如果把这些差异都堆在 Agent 核心，后续加第三个平台会很痛苦。

## 做法

核心思路是：**定义统一消息信封，适配器负责收发，路由器负责分发，Agent 核心只处理信封和工具调用。**

### 1. 统一消息信封

先定义一个内部结构，所有平台都转换成它：

```json
{
  "platform": "telegram",
  "message_id": "123456",
  "chat_id": "-100123456789",
  "user_id": "987654321",
  "text": "/summary",
  "reply_to": null,
  "attachments": [],
  "raw": {}
}
```

字段保持最小集合，新增平台时再扩展，但不要直接暴露平台原始对象。

### 2. 适配器层

为每个平台写一个 adapter，实现同样的接口：

- `connect()` / `disconnect()`
- `send(envelope, text)`
- `edit(envelope, text)`
- `react(envelope, emoji)`
- `ack(envelope)`（可选，用于标记已读或 typing）

Telegram adapter 可以用 `python-telegram-bot` 的 `Application` 处理长轮询；Discord adapter 用 `discord.py` 的 `Client` 处理 Gateway。两边收到消息后，都转换成信封，推给路由器。

### 3. 路由器 / Dispatcher

路由器负责：

- 根据 `platform + chat_id` 做会话隔离，避免不同平台的同名 chat 互相污染。
- 做命令解析，统一前缀 `/`，但允许平台自定义别名（例如 Discord 里 `!` 也支持）。
- 做幂等去重：用 `platform + message_id` 作为幂等键，防止 webhook 重试或 Gateway 重放导致重复执行。
- 将信封交给 Agent 核心处理，并把返回结果交给对应 adapter 发送。

### 4. Agent 核心

Agent 核心只接受信封，输出文本或工具调用结果。它不 import 任何平台 SDK，也不关心消息是从 TG 还是 Discord 来的。工具调用通过 MCP 暴露，比如搜索、数据库查询、文件读取等。这样同一个 Agent 逻辑可以同时服务两个平台，也可以在未来接入第三个。

### 5. 配置与权限

建议用环境变量或配置文件管理：

- 每个平台的 token、webhook secret
- 允许接入的 chat/channel 白名单
- 用户 ID 到内部角色的映射
- 每个平台是否允许执行敏感工具

权限控制在路由器或适配器层做，不要下沉到 Agent 核心。

## 踩坑点

1. **消息长度**  
   Discord 2000 字符限制很容易触发，需要在发送前按段落或句子分割。Telegram 4096 虽然宽裕，但长文本会降低可读性，建议统一按 1800-1900 字符拆条。

2. **回复语义**  
   Telegram 的 `reply_to_message_id` 与 Discord 的 `message_reference` 结构完全不同。在信封里保留 `reply_to` 字段，由 adapter 翻译成各自格式。不要直接传平台对象。

3. **限流**  
   Discord 的 HTTP API 有全局 rate limit，Gateway 的 identify 不能频繁调用。Telegram 的 flood control 对机器人发消息频率敏感。实现一个简单的本地队列 + 指数退避，能避免大多数 429。

4. **重复事件**  
   如果同时启用 webhook 和长轮询，或者 Discord 断线重连，可能收到重复消息。务必在路由器层做幂等：收到相同 `platform + message_id` 的事件直接丢弃。

5. **身份映射**  
   Telegram 用户 ID 是数字，Discord 是 snowflake 字符串。不要直接比较，统一转成字符串，并建立 `platform:user_id` 到内部主体 ID 的映射表。

6. **富文本差异**  
   Telegram 支持 MarkdownV2/HTML，Discord 支持 Markdown 子集。如果 Agent 输出包含特殊字符，建议统一用纯文本发送，或写一个简单的 sanitizer 处理平台不支持的格式。

## 可复用建议

- 先跑通两个平台的最小闭环，再抽象信封和 adapter。过早抽象容易设计出用不上的字段。
- 把幂等键 `platform + message_id` 作为路由器的第一道防线。
- 发送队列放在 adapter 内部，不要让 Agent 核心关心限流。
- 使用 MCP 工具代替直接在 Agent 里调用平台 API，这样工具可以被多个平台复用，也方便测试。
- 给 adapter 层加结构化日志，记录收发的关键字段，排查跨平台问题时能快速定位。
- 如果只是做通知推送，不需要完整 Agent，可以用更轻量的 webhook 转发；但如果是交互式命令，才值得引入路由层。

## 总结

跨平台消息路由并不复杂，关键是拆出“信封、适配器、路由器、Agent 核心”四层。适配器负责平台差异，路由器负责分发与幂等，Agent 核心保持无平台状态。这样同一个 Agent 可以较稳定地同时服务 Telegram 和 Discord，后续接入新平台也只需新写一个 adapter。不要试图在 Agent 核心里处理平台细节，否则维护成本会随平台数量线性上升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ac65eed5065bd6d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/fd2ae050a0d3d0c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f583e22968d76ff0.png)

