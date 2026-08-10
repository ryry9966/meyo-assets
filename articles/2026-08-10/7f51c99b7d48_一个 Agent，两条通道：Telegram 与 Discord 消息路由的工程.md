---
title: 一个 Agent，两条通道：Telegram 与 Discord 消息路由的工程化实践
feedId: 32339
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在日常的 Agent 开发中，一个普遍需求是：同一个业务逻辑机器人，需要同时出现在 Telegram 和 Discord 两个社群中。如果分别为每个平台写一套轮子，不仅维护成本翻倍，随着功能迭代，行为不一致的风险也会急剧上升。

OpenClaw 生态提供了一套可组合的 Agent 管线，但默认只暴露了 HTTP/WebSocket 通道，社区插件市场里也没有现成的“多平台消息路由”方案。笔者近期正好需要让一个基于 OpenClaw 的问答 Agent 同时接入 TG 和 DC，于是顺藤摸瓜地实现了一套轻量的消息路由层。这篇文章记录过程中的设计、步骤、踩坑与可复用组件。

## 问题拆解

要让一个 Agent 同时服务两个平台，本质挑战如下：

- **协议差异**：Telegram 通过 getUpdates 长轮询或 Webhook 推送 `Update`，Discord 通过 Gateway 或 HTTP Interaction。消息载荷结构完全不同。
- **鉴权体系**：TG 用 Bot Token，DC 用 Bot Token + OAuth2（Interaction 需验证签名）。
- **消息语义差异**：TG 支持 MarkdownV2 / HTML，DC 使用自己的一套 markdown 子集；TG 一条消息一个 chat_id，DC 多线程（channel/thread）且有交互组件（按钮、下拉菜单）。
- **会话管理**：两个平台的用户标识、群组标识不通用，Agent 需要统一上下文，例如按“平台+聊天ID”隔离会话。

如果我们能在 Agent 入口前加一层适配器，把“平台消息”转化为统一的 `UserMessage`，再把 Agent 输出的 `AgentResponse` 转回平台可发送的格式，那么核心管线完全无感知差异。

## 核心设计

我们抽象出三个关键组件：

```
Platform Adapter → Normalized Message → OpenClaw Agent → Normalized Response → Platform Adapter
```

- **Normalized Message**：包含 `platform`, `chat_id`, `user_id`, `text`, `attachments`, `message_id`, `reply_to` 等字段，格式化的纯文本内容，不含任何平台富文本。
- **Platform Adapter**：负责接收来自特定平台的事件，转换成 Normalized Message，调用 Agent 管线，拿到 Normalized Response后再转换回平台格式并发送。

这种设计的好处是：新增一个平台只需实现一个新的 Adapter，Agent 无需任何修改。同时，可以在 Normalized Message 中注入平台元信息（如来源频道名称），供日志或路由策略使用。

## 实践步骤

### 1. 环境准备

假设已有一个可运行的 OpenClaw 实例，Agent 已能处理 `UserMessage` 并返回 `AgentResponse`。我们使用 Python 3.11，依赖：`python-telegram-bot`（v20+ 异步版本）和 `discord.py`（v2.4+，支持 Interaction）。

```bash
pip install openclaw python-telegram-bot discord.py
```

### 2. 定义统一消息模型

在 `core/messages.py` 中：

```python
from dataclasses import dataclass
from typing import Optional, List

@dataclass
class NormalizedMessage:
    platform: str          # "telegram" or "discord"
    chat_id: str           # 平台内的对话唯一标识
    user_id: str           # 发送者ID
    text: str              # 已清洗的纯文本
    attachments: List[str] # 文件URL列表
    message_id: str        # 原始消息ID，用于回复定位
    reply_to: Optional[str]=None
```

Agent 输出同样使用一个归一化的 Response 对象。

### 3. 实现 Telegram Adapter

利用 `python-telegram-bot` 的 `Application`，监听 `MessageHandler(filters.TEXT & ~filters.COMMAND, callback)`。

回调中获取 `update.effective_chat.id` 作为 `chat_id`，`update.effective_user.id`，`update.message.text`，然后组装成 NormalizedMessage 交给 Agent。Agent 返回后，用 `update.message.reply_text` 发送回复，注意转义 MarkdownV2 特殊字符。

```python
async def tg_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    norm_msg = NormalizedMessage(
        platform="telegram",
        chat_id=str(update.effective_chat.id),
        user_id=str(update.effective_user.id),
        text=update.message.text or "",
        attachments=[],
        message_id=str(update.message.message_id)
    )
    response = await agent.process(norm_msg)
    await update.message.reply_text(response.text, parse_mode=None) # 使用纯文本避免转义问题
```

### 4. 实现 Discord Adapter

Discord Interaction 方式推荐使用 `discord.py` 的 `commands.Bot` 监听 `on_message` 事件（或改用 `app_commands` 处理 slash 命令）。这里以普通消息为例，监听 `on_message`，过滤掉机器人自身消息，提取 `message.channel.id` 作为 `chat_id`（可结合 `guild.id` 保证全局唯一），`message.author.id` 作为 `user_id`，内容直接用 `message.content`。回复使用 `message.channel.send()`。

注意 Discord 的 Markdown 与常用 Markdown 略有差异，建议在 Adapter 层将 Agent 输出转换为 Discord 支持的格式（如粗体 `**` 正常，但标题 `#` 不支持，需要降级处理）。

```python
@bot.event
async def on_message(message):
    if message.author == bot.user:
        return
    norm_msg = NormalizedMessage(
        platform="discord",
        chat_id=str(message.channel.id),
        user_id=str(message.author.id),
        text=message.content,
        attachments=[a.url for a in message.attachments],
        message_id=str(message.id)
    )
    response = await agent.process(norm_msg)
    # 简单清理 Discord 不支持的 Markdown 元素（例如将 # 替换为 **粗体**）
    cleaned = response.text.replace('# ', '**') + '**' if '# ' in response.text else response.text
    await message.channel.send(cleaned)
```

### 5. 集成到 OpenClaw Agent

将两个 Adapter 实例化并在独立异步任务中启动。OpenClaw 的 `Agent` 本身就是异步可调用对象，天然支持并发，无需额外锁。两个 bot 的事件循环可以各自运行，只需注意 Telegram bot 的 `Application.run_polling()` 和 Discord bot 的 `bot.run(token)` 会阻塞当前事件循环。通常使用 `asyncio.gather` 将两个任务并行启动：

```python
async def main():
    await asyncio.gather(
        run_tg_bot(),
        run_dc_bot()
    )
```

## 踩坑记录

- **Telegram Markdown 转义地狱**：Agent 输出可能含特殊字符 `*_[]()~` 等。如果不用 `parse_mode` 或设为纯文本，可避免大部分问题。如需富文本，建议在 Adapter 内实现一个白名单转义函数，只对需要的标记转义，其余保持原貌。
- **Discord 消息长度限制**：单条消息上限 2000 字符，超出自动分段。需在 Adapter 中检查长度并按首句或段落分片发送，否则 API 抛出异常。
- **附件处理**：TG 图片、文件会以 `Update` 的不同字段出现（`photo`、`document`），需要处理为统一附件列表。Discord 的附件可直接取 URL，但注意过期时间和大小限制。
- **用户上下文隔离**：Agent 如果使用内存存储会话，`chat_id` 必须全局唯一。可以将 `f"{platform}:{chat_id}"` 组合作为会话键，避免 TG 和 DC 偶然的 ID 碰撞。
- **速率限制**：两个平台都有 rate limit，尤其是 Discord 全局 50 请求/秒 和单频道限频。建议在 Adapter 内加入简单的令牌桶或使用 `discord.py` 内置的自动重试机制；TG 侧需注意返回 `RetryAfter` 异常并休眠。

## 可复用建议

- **抽象 Adapter 接口**：定义一个 `BaseAdapter`，包含 `start()`, `send_message(chat_id, NormalizedResponse)` 抽象方法，所有平台实现该接口。未来扩展 Slack、WhatsApp 等只需新增 Adapter。
- **消息队列解耦**：若 Agent 处理耗时较长（>3秒），可以将 NormalizedMessage 投递到 Redis Stream，由 Worker 消费，避免 Webhook 超时。Adapter 只负责入队和送答。
- **统一日志与监控**：在 NormalizedMessage 转发的切面记录 platform、chat_id、处理耗时，可轻松构建跨平台的监控面板。
- **兜底策略**：当 Agent 不可用或处理超时，Adapter 应返回预设的“服务暂不可用”文案，避免用户看到未回复的空白。

## 总结

通过一层薄薄的消息归一化层，我们成功让同一个 OpenClaw Agent 同时活跃在 Telegram 和 Discord 中，且后续功能迭代完全共享。整个 Adapter 层代码不超过 300 行，但解决了协议差异、格式转换、会话隔离等关键问题。这种做法在实际工程中可扩展性极强，只需关注平台特有的“方言”转译，核心管线保持纯粹。

如果你的 Agent 也需要多平台覆盖，不妨试试这套轻量路由方案。

---

