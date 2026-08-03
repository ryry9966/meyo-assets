---
title: 跨平台消息路由实战：让一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 31425
source: 综合讨论
publishedAt: 2026-08-03
---

# 跨平台消息路由实战：让一个 OpenClaw Agent 同时服务 Telegram 和 Discord

## 背景

在自动化实践中，我们常遇到一个尴尬场景：辛苦调好的 Agent 只挂在 Telegram 上，但社区、客户、同事又活跃在 Discord。难道要维护两个 Agent 实例、两套配置、两份上下文？这不仅增加运维负担，更会导致会话数据割裂。

OpenClaw 的插件体系和 MCP 扩展能力，提供了另一种思路：**一个 Agent 核心，多平台适配**。本文记录如何让同一个 OpenClaw Agent 实例同时接入 Telegram 和 Discord，实现消息统一路由、上下文隔离、原生回复。

## 问题拆解

跨平台消息路由本质上要解决三个问题：

1. **接入异构协议**：Telegram Bot API 与 Discord Gateway 的认证、事件机制、消息格式完全不同。
2. **消息模型归一化**：两个平台的文本、图片、按钮等表达方式差异巨大，需要映射为 Agent 能理解的统一结构。
3. **会话与上下文管理**：同一用户在不同平台的身份不同，必须保证 Agent 记住的上下文不会串，且回复能准确返回到对应平台。

## 实现路径

我们基于 OpenClaw 现有机制，采用 **适配器 + 路由层** 的轻量架构。

**第一步：创建统一消息 Schema**

定义一个平台无关的消息对象，核心字段：

- `platform`: `"telegram"` 或 `"discord"`
- `userId`: 平台内用户唯一标识
- `sessionId`: 由 `platform + userId` 生成的隔离会话 ID
- `content`: 归一化后的文本
- `rawEvent`: 保留原始事件，供特定平台功能回退

这一步建议做成 MCP 资源或 OpenClaw 插件中的类型定义，便于后续维护。

**第二步：实现平台适配器**

- Telegram 侧使用 `python-telegram-bot`，通过长轮询获取 update。每个消息构造为统一 Message 对象，调用 Agent 的入口函数。
- Discord 侧使用 `discord.py`，开启 Message Content Intent，监听 `on_message` 事件，同样转为统一 Message。

将两个适配器分别实现为 OpenClaw 的独立插件，这样可按需启停，不会相互干扰。

**第三步：构建路由与调度**

路由层接收来自两个插件的 Message，根据 `platform + userId` 查找或创建会话上下文。OpenClaw 的 Agent 上下文本身就是按会话隔离的，只需确保 `sessionId` 一致即可。

Agent 处理完成后，回调函数中根据 `platform` 字段判断回复路径：
- Telegram：通过 `context.bot.send_message(chat_id=...)` 发送
- Discord：通过 `message.channel.send(...)` 发送

为避免回复风暴，设置每次触发只回复一条核心消息，附件、图片等通过后续指令扩展。

## 实际踩坑记录

### Telegram 长轮询与 Webhook 的选择
长轮询开发方便，但生产环境建议 Webhook，减少资源占用。但如果用一条 Agent 实例同时处理两个平台，Telegram 用 Webhook 时要注意端口冲突，推荐反向代理统一转发。

### Discord Intent 与权限
忘记在 Discord Developer Portal 启用 Message Content Intent 会导致 `on_message` 收不到消息。此外，私信与服务器频道权限不同，测试时要分别验证。

### 消息去重与防 loop
一次用户输入可能触发多个事件（如编辑、反应），适配器务必过滤，只处理全新文本消息。同时要绝对禁止 Agent 触发自身的回复再次传入，否则会陷入无限对话循环。实现中加入了“忽略来自 bot 自身的消息”的判断。

### 平台格式差异
Telegram 支持 MarkdownV2，Discord 使用子集略有不同。统一回复时采用纯文本 + 最少公共标记，避免格式错乱。如需富文本，建议在适配器中单独实现转换函数。

### 上下文记忆隔离
起初直接使用同一 `userId` 会导致跨平台串记忆 —— 比如 Discord 用户 ID 是数字，Telegram 也是数字，相同 ID 但代表不同人。解决方案是强制 session 使用 `platform + userId` 组合字符串，彻底隔离。

## 可复用建议

1. **抽象平台接口**：定义 `PlatformAdapter` 基类，包含 `start()`, `stop()`, `send_message()` 方法，任何新平台只需实现该接口。
2. **消息队列化**：适配器将消息放入队列，由 Router 统一消费，不仅解耦，也能自然实现速率控制。
3. **配置驱动**：平台 Token、通道白名单等通过 OpenClaw 环境变量或配置文件注入，避免硬编码。
4. **监控与告警**：记录每个平台的消息处理成功率、延迟，尽早发现单平台故障。

## 总结

通过一个轻量的路由层，我们成功让同一个 OpenClaw Agent 具备横跨 Telegram 和 Discord 的服务能力。这种方式不仅减少了资源浪费，更让用户无论从哪个平台接入，都能获得一致的上下文体验。随着 MCP 生态成熟，未来甚至可以将适配器封装为标准 MCP Server，让 Agent 的“跨平台”变得像安装插件一样简单。

---

