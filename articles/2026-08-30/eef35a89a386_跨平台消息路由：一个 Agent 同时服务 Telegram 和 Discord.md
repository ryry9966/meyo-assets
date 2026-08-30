---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord 的工程实践
feedId: 35429
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

很多 Agent 实践是从单一平台起步的：先跑通 Telegram bot，再想扩展到 Discord。如果直接复制一份代码、部署两个实例，短期能工作，但很快会遇到状态割裂、配置重复、工具调用上下文不一致的问题。更合理的做法是让一个 Agent 实例同时服务多个平台，通过统一消息路由处理来自 Telegram 和 Discord 的输入。

这篇文章记录我在 OpenClaw 生态下做跨平台消息路由的工程化实践，重点不是“能跑”，而是后续维护成本可控。

## 问题

Telegram 和 Discord 的 API 差异很大，主要体现在：

- 消息长度限制不同：Telegram 单条上限 4096 字符，Discord 2000 字符。
- Markdown 方言不同：Telegram 支持 MarkdownV2/HTML，Discord 支持自己的 Markdown 子集。
- 附件处理不同：Telegram 依赖 file_id 下载，Discord 附件 URL 有时效性。
- 用户标识不同：Telegram user_id 和 Discord user_id 没有任何关联。
- 速率限制规则不同：Discord gateway 有 5/5s 限制，Telegram 有 flood control。

如果直接在一个 handler 里写满 `if platform == "telegram"` 的判断，代码会迅速腐化。所以需要一层抽象。

## 做法/步骤

### 1. 定义统一消息模型

先抽象一个平台无关的消息结构，所有平台适配器都转换成这个结构：

```python
@dataclass
class InboundMessage:
    platform: str          # "telegram" | "discord"
    chat_id: str
    user_id: str
    text: str
    attachments: list
    reply_to_message_id: str | None
```

出站消息类似：

```python
@dataclass
class OutboundMessage:
    target_platform: str
    chat_id: str
    text: str
    attachments: list
    reply_to: str | None
```

### 2. 平台适配器

每个平台写一个适配器，负责两件事：接收平台事件并转换为 `InboundMessage`，以及把 `OutboundMessage` 转换回平台原生格式发送。

- Telegram 适配器：用 `aiogram` 或 `python-telegram-bot` 处理 update，提取 `chat_id`、`user_id`、`text`、附件 file_id。
- Discord 适配器：用 `discord.py` 或 `pycord` 监听 message 事件，提取 `channel_id`、`author.id`、`content`、附件 URL。

适配器内部可以保留平台特有逻辑，但对外只暴露统一接口。

### 3. 路由层与 Agent 解耦

Agent 核心只依赖 `InboundMessage` 和 `OutboundMessage`，不关心消息来自哪个平台。路由层负责：

- 根据 `platform + chat_id` 维护会话键。
- 把 Agent 返回的 `OutboundMessage` 路由到对应适配器发送。
- 如果需要跨平台用户绑定，通过数据库维护 `telegram_user_id` 和 `discord_user_id` 的映射。

### 4. 部署与并发

单进程同时运行两个适配器，推荐用 `asyncio` 事件循环。如果使用 webhook 模式，需要两个 HTTP 端点分别接收 Telegram 和 Discord 的回调；如果使用长轮询/gateway，可以放在同一个事件循环里。

建议引入一个队列（`asyncio.Queue` 或 Redis Stream）解耦接收和处理。接收回调只做消息入队，Worker 从队列取消息、调用 Agent、发送回复。这样即使 Agent 处理某个消息耗时较长，也不会阻塞其他平台的消息接收。

## 踩坑点

### 消息长度分段

最简单的按字符切分会导致 Markdown 代码块被截断。建议实现安全分段：优先按段落切，代码块或围栏块保持完整，实在超长再按行切，并在每段末尾加上续接标记。

### Markdown 转换

不要试图做一个“万能 Markdown 转换器”。Telegram 的 MarkdownV2 需要转义 `.`、`-`、`!` 等字符，Discord 的支持子集又不完全一样。建议统一用纯文本，或者为每个平台单独维护格式化函数，接受降级。

### 附件时效

Discord 附件 URL 会在几小时后过期，Telegram 文件需要通过 `getFile` API 获取下载链接。跨平台转发媒体时，必须先将文件下载到本地或对象存储，再上传到目标平台，不能直接传 URL。

### 用户身份映射

两个平台的 user_id 互不相关。如果需要跨平台识别同一用户（例如在 Telegram 绑定 Discord 账号），必须维护绑定表。如果没有这个需求，就明确说明“跨平台不共享会话”，避免用户混淆。

### 防止消息循环

如果 Telegram 和 Discord 之间通过桥接机器人连接，Agent 可能会收到来自桥接 bot 的消息，造成循环。需要识别桥接 bot 的 ID 并忽略，或者对消息指纹（platform + message_id）做去重。

### 速率限制与长任务

发送失败要退避重试，不要立即重抛。Agent 调用工具可能耗时（搜索、生成图片等），不能在消息接收回调里同步等待，应该先返回一个“处理中”状态，然后在后台任务完成后更新消息。

## 可复用建议

- **适配器模式是核心**：保持平台代码与 Agent 逻辑隔离，新增平台时只写一个新适配器。
- **使用队列解耦**：接收和处理分离，统一错误处理和重试策略。
- **每个平台独立 bot token**：测试时用独立 bot，避免污染生产环境。
- **记录全局唯一消息 ID**：`platform + message_id` 可用于追踪、去重和幂等。
- **为每个平台写健康检查**：监控连接状态，网关断开时能自动重连。

## 总结

跨平台消息路由不是简单地把两个 bot 塞进一个进程，而是通过统一消息模型和适配器模式隔离平台差异。工程上重点处理消息长度、格式转换、身份映射和限流。这套结构在 OpenClaw 生态里同样适用，保持 Agent 只关心业务逻辑，平台差异由适配器消化。后续维护成本会低很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bde6924ca643f3fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bdf7d84731f76514.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/407972ecaf9ff6ec.png)

