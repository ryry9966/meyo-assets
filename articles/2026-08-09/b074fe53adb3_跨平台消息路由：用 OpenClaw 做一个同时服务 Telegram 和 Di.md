---
title: 跨平台消息路由：用 OpenClaw 做一个同时服务 Telegram 和 Discord 的 Agent
feedId: 32237
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在社区里，不少做自动化、Bot 开发的朋友会遇到一个场景：业务逻辑是同一套，但需要同时覆盖 Telegram 和 Discord 两个用户群。用两个独立的 Agent 分别维护，不仅浪费算力，还会引入状态不一致、代码重复、配置分离等问题。

OpenClaw 提供了「一个 Agent 大脑 + 多个工具/插件」的架构，正好适合这类需求。配合 MCP 或自定义插件，我们可以让同一个 Agent 同时处理来自 Telegram 和 Discord 的消息，并根据平台特性做适配回复。

## 问题拆解

关键挑战不是「能不能同时监听两个平台」，而是：

- 消息收发模型不同（Telegram 是长轮询/Webhook，Discord 是 Gateway/Webhook）
- 消息格式与限制差异大（Markdown 语法、长度上限、文件处理）
- 用户身份与上下文管理需要统一抽象
- 速率限制和错误重试策略要区分平台

目标：让 Agent 只关心「收到什么消息、该做什么、回复什么内容」，平台差异全部封装在适配层。

## 整体方案

架构如下：

- OpenClaw Agent：负责推理、调用工具、生成回复
- 平台适配器（Adapter）：负责接收消息、转换成标准 Message，以及将 Agent 回复转换成平台能发的格式
- 通道（Channel）：负责具体与平台 API 通信（发送消息、处理 Webhook）

我选择用 Python 实现，利用 `python-telegram-bot` 和 `discord.py` 两个成熟的库，再通过自定义 MCP 工具暴露给 OpenClaw。当然，你也可以直接用 HTTP Webhook 加请求，不需要额外库。

### 1. 统一消息模型

无论来自哪里，内部统一用下面这个结构：

```python
@dataclass
class PlatformMessage:
    platform: str          # "telegram" | "discord"
    channel_id: str        # 群组/频道 ID
    user_id: str           # 发送者 ID
    text: str
    attachments: list[str] # 文件 URL
    raw: Any               # 原始事件
```

### 2. Telegram 端适配

Telegram Bot 通过 Webhook 收到 update，适配器将其包装成 `PlatformMessage`，然后调用 Agent 处理。回复时，需要将 Agent 返回的文本转换成 Telegram 支持的 MarkdownV2 并处理长度分段。

伪代码：

```python
async def handle_telegram_update(update):
    msg = adapt_tg_to_message(update)
    reply_text = await agent.process(msg)
    if len(reply_text) > 4096:
        chunks = split_text_for_tg(reply_text)
    else:
        chunks = [reply_text]
    for chunk in chunks:
        await bot.send_message(chat_id=msg.channel_id, text=chunk, parse_mode='MarkdownV2')
```

### 3. Discord 端适配

Discord 通过 Gateway 或 Webhook 收到 Message 事件，同样是转换后交给 Agent。回复需要注意 Discord 的 2000 字符限制，以及 embed 的使用。较长的回复可以拆成多条普通消息，或使用 embed 的 `description` 字段（同样有长度上限）。

### 4. Agent 注册为 MCP 工具

如果你使用 OpenClaw 的 MCP 插件系统，可以将「发送消息」抽象成 MCP 工具，工具内部根据 `platform` 参数转发到对应的平台 SDK。这样 Agent 根本不知道 Telegram 和 Discord 的区别，只需要调用 `send_message` 工具：

```json
{
  "tool": "send_message",
  "arguments": {
    "platform": "telegram",
    "channel_id": "...",
    "text": "..."
  }
}
```

### 5. 路由与并发

同一 Agent 实例同时服务两个平台，需要考虑并发处理。这里我用 `asyncio.gather` 分别运行两个平台的长轮询或 Webhook 服务，Agent 本身可以做线程安全或者利用异步队列串行化决策部分，避免状态竞争。

## 踩坑记录

- **消息格式转换**：Telegram 的 MarkdownV2 要求对 `_ * [ ] ( ) ~ > # + - = | { } . !` 全部转义。Discord 的 Markdown 虽然宽松，但不支持嵌套，代码块里的特殊符号也可能被解析。建议先生成纯文本，再根据平台做轻度格式化，不要贪图花哨。
- **长度限制**：Telegram 单条 4096，Discord 单条 2000。务必在回复前按段落或句子切割，避免截断在链接或代码块中间。可以写一个通用的 `split_text` 函数，优先按换行切，再按空格补刀。
- **速率限制**：Discord 有全局速率限制，Telegram 对群组也有 20 条/分钟的限制。Agent 如果突然批量回复（例如欢迎机器人轰炸），会被临时封禁。一定要实现指数退避和消息队列缓冲。
- **用户身份映射**：如果要在两个平台间做用户关联，需要额外数据库维护 `telegram_id` <-> `discord_id` 的映射。这不是 Agent 的职责，但要在适配层提供接口。
- **错误恢复**：Webhook 连接断开、Gateway 重连需要自动恢复。Discord 的 `discord.py` 自带重连，但自定义 Webhook 实现要自己处理状态。

## 可复用建议

把这个工程抽象成三个层次，以后加 WhatsApp、Slack 都很容易：

1. **Channel**：封装平台 API 的发送与接收，暴露 `on_message` 回调
2. **Adapter**：实现消息模型转换、格式清洗、长度分割
3. **Agent Core**：只依赖统一 Message 和工具调用，不碰平台细节

建议把每个平台的 Adapter 写成独立模块，通过配置启用，Agent 启动时动态加载。这样即便某个平台 SDK 有安全漏洞需要升级，也不会影响核心逻辑。

## 总结

让一个 OpenClaw Agent 同时服务 Telegram 和 Discord 完全可行，关键在于做一层干净的平台抽象。核心工作量不在 Agent 本身，而在适配层的格式处理与错误重试。一旦这套架子搭好，后续扩展新平台边际成本很低。

如果你已经有一个跑在单平台上的 Agent，改造成跨平台大概需要 1-2 天的工程时间，大部分会花在处理各平台边角限制的调试上。希望这篇方案能帮你少走弯路。

---

