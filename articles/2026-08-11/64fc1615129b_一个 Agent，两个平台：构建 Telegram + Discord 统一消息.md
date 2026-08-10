---
title: 一个 Agent，两个平台：构建 Telegram + Discord 统一消息路由
feedId: 32469
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在构建基于 OpenClaw 的 Agent 时，一个很常见的需求是让同一个 Agent 逻辑同时服务于多个即时通讯平台，最常见的组合就是 Telegram 与 Discord。如果为每个平台单独维护一套接入代码，不仅导致逻辑重复，还会让维护成本翻倍，且难以保证行为一致性。更理想的做法是设计一套统一的消息路由层，让 Agent 只关心“收到了什么消息、该回复什么”，而完全不必感知消息来自哪个平台。

本文记录了一次实际工程实践：如何在不侵入 Agent 核心逻辑的前提下，让一个 OpenClaw 实例同时处理来自 Telegram 和 Discord 的消息，并保证上下文、命令、错误处理等工程化细节。

## 问题分析

表面上看，让 Agent 同时接入两个平台无非是多启动两个 bot 而已。但实际落地时，下面几个问题会迅速暴露：

1. **消息模型不统一**：Telegram 使用 `chat_id` + `message_id`，Discord 使用 `channel_id` + `message_id`，且数据结构差异巨大（附件、嵌入、按钮等）。
2. **会话上下文绑定**：同一个用户可能在不同平台与 Agent 交互，是否需要隔离？通常应根据平台+用户维度分别维护会话状态。
3. **命令分发机制**：两个平台都可能有命令前缀（如 `/`），但也可能需要支持自然语言触发，如何让 Agent 统一处理。
4. **回复路由与格式适配**：Agent 输出的内容是结构化的（文本、embed、键盘、文件），需要根据目标平台的能力进行适配和裁剪。
5. **并发与幂等**：两个平台的 webhook 或长轮询可能同时将消息送入 Agent，需要考虑消息处理的并发安全和重复消息去重。

单一的 Agent 实例如果直接暴露给两个平台的原生接口，代码很快就会变成平台特性的堆砌。解决思路是引入一个**消息抽象层**，将平台差异封装在适配器中，Agent 只与统一的消息对象交互。

## 设计方案

整体架构分为三层：

- **平台适配层（Adapters）**：负责与 Telegram Bot API / Discord Gateway (或 HTTP) 交互，接收原始事件，转换成统一消息模型，并将最终回复转换回平台可发送的格式。
- **消息总线（Message Bus）**：一个轻量级的事件队列（可以是内存 channel 或 Redis Stream），所有适配器将统一消息推入，Agent 处理完毕后将回复推到对应的发送队列。
- **Agent 核心（OpenClaw Pipeline）**：接收统一消息，进行语义理解、工具调用、记忆管理等，返回抽象回复对象。

统一消息模型核心字段设计如下：

```python
class UnifiedMessage:
    platform: str          # "telegram" 或 "discord"
    channel_id: str        # 平台特有的会话标识
    user_id: str           # 跨平台唯一用户 ID（可加前缀区分）
    message_id: str        # 用于去重和定位原始消息
    text: str
    attachments: list[Attachment]
    raw_payload: dict      # 保留原始数据以备不时之需
```

回复模型同样抽象：

```python
class UnifiedReply:
    text: str | None
    embeds: list[Embed] | None
    inline_keyboard: list[...] | None
    files: list[File] | None
    target_message_id: str | None  # 用于回复/编辑
```

## 具体实现步骤

### 1. 平台适配器实现

以 Telegram 为例，使用 `python-telegram-bot` 处理 webhook，在回调中将 `update` 转换为 `UnifiedMessage`，推入消息总线。Discord 端可使用 `discord.py` 的 `on_message` 事件，同样构造统一消息。

关键点：适配器需要负责为每个用户生成**稳定的唯一用户 ID**，例如 `tg:` + Telegram user ID，`ds:` + Discord user ID，确保不同平台的同一自然人被视为不同用户（绝大多数场景下应隔离，除非有跨平台身份绑定）。

### 2. 消息总线与并发处理

使用 `asyncio.Queue` 作为内部总线，Agent 以协程方式逐个消费消息。如果场景需要多实例扩展，可以将队列外移至 Redis。每条消息的处理结果带上一个回调协程，让适配器可以调取回复并发送。

为了避免同一用户同一消息被重复处理（网络重试、webhook 重放），可以引入基于 `message_id` 的幂等去重，比如用 Redis 做短期缓存记录。

### 3. Agent 侧统一处理

OpenClaw 的 `MessagePipeline` 本身就接收标准化的 `Message` 对象，我们可以直接复用，只需要保证 `platform` 等元数据作为 context 的一部分传入。对于命令路由，可以在 Prompt 中明确告知 Agent 当前平台，让其输出符合平台限制的内容（比如 Discord 支持 embed 而 Telegram 更适合 MarkdownV2）。

### 4. 回复适配输出

Agent 输出抽象 `UnifiedReply`，由每个平台的发送适配器负责转换。例如 Telegram 的 `reply_to_message_id` 可以通过 `target_message_id` 实现，Discord 的 embed 则只在平台支持时才填充。如果 Agent 输出了一条长文本，Telegram 发送适配器可以自动切分多条消息，而 Discord 则可保持原样。

## 踩坑与排障

- **Telegram 长轮询与 webhook 选择**：开发环境用长轮询方便，但生产环境必须用 webhook 并处理好 SSL 证书。同时 webhook 超时设置过短会导致消息积压，建议设置合理的超时并启用异步 worker。
- **Discord rate limit**：Discord 对消息发送有严格全局限流，尤其是 embed 和文件上传。适配器必须内置指数退避和重试队列，否则很容易触发 429 导致消息丢失。
- **消息编辑同步**：用户在原平台编辑消息，Agent 如果已经处理并回复，修改后的消息是否需要重新触发？一般情况下不建议主动同步，但至少要在去重逻辑中避免重复处理编辑消息（Discord 的 `on_message_edit` 会再次触发）。
- **附件处理**：Telegram 和 Discord 的附件链接均为临时有效，下载时需要带认证 token。Agent 如果需要处理图片、文件，统一消息层应该提前转存到可公开访问的存储（如对象存储），并传递稳定 URL。
- **安全与鉴权**：防止其他 bot 误接入或恶意消息注入，适配器层应校验来源 token 和签名。

## 可复用建议

1. **抽象接口先行**：将 `Adapter` 定义为一个协议类，要求实现 `start`, `send`, `stop` 等方法，新增平台只需实现该接口即可接入统一路由。
2. **会话存储外置**：使用 Redis 保存用户上下文（Platform + UserID 为 key），Agent 每次处理时从 Redis 加载并更新，避免单实例内存无法持久化。
3. **配置集中管理**：每个平台的 token、webhook 地址、功能开关等均通过配置文件或环境变量注入，不与代码耦合。
4. **充分利用 OpenClaw 的 Pipeline 能力**：在 Pipeline 中根据 `platform` 字段动态调整 System Prompt，或在预处理阶段注入平台特有的格式说明，让模型自行适配输出格式。

## 总结

跨平台消息路由的实质是做一层**平台抽象**，让 Agent 与通信渠道解耦。一旦这套机制搭建完成，后续接入 WhatsApp、Slack 等平台成本极低。工程落地的关键在于：统一消息模型设计合理，适配器要足够“脏活”地将平台差异封装好，同时要重视并发去重和限流这些看似琐碎但线上致命的问题。对于已经在使用 OpenClaw 的团队，花一两天时间搭建这套路由层，带来的长期收益远超预期。

---

