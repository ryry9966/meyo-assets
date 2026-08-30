---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 35347
source: 综合讨论
publishedAt: 2026-08-30
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

在 OpenClaw 这类可扩展 Agent 框架里，很多自托管机器人一开始只接一个平台，比如 Telegram。但当同一个运维助手、客服机器人或社区问答 Agent 需要同时服务 Discord 和 Telegram 两个社区时，如果每个平台单独跑一个实例，会出现几个问题：同一套任务逻辑被复制多份，状态和记忆无法合并，配置漂移，后续维护成本翻倍。

更合理的做法是让一个 Agent 核心同时接两个平台。本文记录一种轻量的跨平台消息路由方案：在 Agent 内或作为插件实现平台适配层，把 Telegram 和 Discord 的消息归一化成统一事件，再交给同一套处理逻辑。

## 问题

两个平台的差异比想象中大：

- **接入方式不同**：Telegram 常用 getUpdates 长轮询或 webhook；Discord 通常用 Gateway WebSocket，按钮交互还要走 Interactions Endpoint。
- **消息结构不同**：Telegram 有 `chat_id`、`message_id`、`entities`；Discord 有 `channel_id`、`guild_id`、`message_id`、`components`、`embeds`。
- **格式不同**：Telegram 支持 MarkdownV2/HTML；Discord 支持 Markdown 子集和嵌入消息。
- **交互组件不同**：Telegram 用 inline keyboard；Discord 用 buttons/select menus。
- **限流策略不同**：Telegram 有每群组每分钟 20 条左右的限制，Discord 有全局 50 req/s 和每路由限流。

如果让业务逻辑直接调用平台 API，后面每接一个新平台都得改核心代码。所以关键是先定义好边界。

## 做法 / 步骤

### 1. 定义统一消息模型

先抽象一层与平台无关的事件和动作。入站事件可以这样设计：

```text
type: text | command | callback | reaction
platform: telegram | discord
sender_id: string
chat_id: string
channel_id: string?
message_id: string
content: string
raw: object
reply_to: object?
```

出站动作则抽象为：

```text
send_text(target, content, reply_to?)
edit_text(target, message_id, content)
send_buttons(target, buttons)
set_typing(target)
```

业务逻辑只依赖这个模型，不感知具体平台。

### 2. 为每个平台实现 Adapter

每个平台一个 Adapter，负责两件事：

- **接收消息**：把平台原生事件转换成上面的统一事件。
- **发送动作**：把统一动作翻译成平台 API 调用。

例如 TelegramAdapter 可以用 webhook 接收 update，DiscordAdapter 通过 Gateway 接收 message create 事件。在 OpenClaw 中，可以把这个 Adapter 做成插件，启动时注册到事件总线。

### 3. 事件总线路由

Adapter 收到原始消息后，做归一化，然后投递到事件总线。Agent 核心从总线消费事件，处理后产生出站动作，再根据动作里的 `platform` 和 `target` 路由回对应 Adapter 执行。

这样做的好处是：Agent 核心完全不 import Telegram 或 Discord 的 SDK。

### 4. 配置化路由规则

不是所有频道都要响应。可以用配置表控制启用范围：

```yaml
platforms:
  telegram:
    enabled: true
    token_env: TG_BOT_TOKEN
    mode: webhook
  discord:
    enabled: true
    token_env: DISCORD_BOT_TOKEN
    intents: [guild_messages, direct_messages]
routing:
  rules:
    - platform: telegram
      allow_chats: ["-100123456789"]
    - platform: discord
      allow_channels: ["123456789012345678"]
      ignore_bots: true
```

这样可以避免机器人在错误的地方插嘴。

## 踩坑点

### 1. 回复上下文丢失

如果用户在 Telegram 和 Discord 上同时发消息，Agent 回复时必须带上对应平台的原生引用，否则用户会分不清回复的是哪条。统一模型里要保留 `reply_to`，并由 Adapter 正确映射回平台 API 的 `reply_parameters`（Telegram）或 `message_reference`（Discord）。

### 2. 按钮回调的响应时限

Discord 的 button interaction 要求 3 秒内响应，否则会显示失败。Telegram 的 callback query 虽然没这么严格，但也要及时 `answerCallbackQuery`。统一抽象成 `callback` 事件后，Adapter 层必须处理 deferred response，避免业务逻辑耗时导致交互超时。

### 3. 消息编辑循环

Telegram 和 Discord 都支持编辑消息。如果 Agent 订阅了编辑事件，并且自己也编辑消息，很容易形成循环。建议只有用户触发的编辑事件才进入处理链，Agent 主动产生的编辑不再次触发。

### 4. 格式转换

Telegram 的 MarkdownV2 和 Discord 的 Markdown 子集不完全兼容，尤其是 escaping 规则。内部最好使用纯文本或一种非常受限的中间格式，由各 Adapter 负责转换成本平台支持的格式。

### 5. 限流与重试

Discord 的 429 会返回 `retry_after`，必须做退避队列。Telegram 群组限流触发后如果不等待，消息会静默丢失。两个平台都要做发送队列和指数退避。

### 6. 幂等

Webhook 和 Interactions Endpoint 都可能重试。以 `message_id` 或 interaction token 做幂等键，避免同一条消息被处理两次。

## 可复用建议

- **Adapter 只做转换，不做业务**：所有判断逻辑放在 Agent 核心，Adapter 保持薄。
- **写 Fake Adapter 用于本地测试**：不依赖真实 Telegram/Discord 网络，模拟消息进和动作出。
- **记录原始 payload 日志**：平台字段经常变动，排查归一化错误时原始 payload 非常有用。注意脱敏。
- **监控每个平台的延迟和失败率**：出站动作的发送成功率、平均延迟、限流等待时间。
- **灰度接入新平台**：先用一个测试频道跑通全链路，再放开到其他频道。

## 总结

一个 Agent 同时服务 Telegram 和 Discord，本质不是多接两个 SDK，而是建立统一消息模型和严格的路由边界。Adapter 层负责处理平台脏活，业务逻辑保持平台无关。这样后续再加 WhatsApp、Slack 或 Matrix，只需要新增一个 Adapter，而不用修改 Agent 核心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/0c66c797d2c2df35.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/1080f2a48e4c29ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c5889e82ef86a50a.png)

