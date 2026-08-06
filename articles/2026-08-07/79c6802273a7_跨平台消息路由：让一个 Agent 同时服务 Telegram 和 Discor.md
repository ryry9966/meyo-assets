---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 31927
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在社区运营和自动化实践里，我们常常需要让同一个助理 Agent 同时出现在 Telegram 群组和 Discord 服务器中。  
如果为每个平台单独部署一个 Agent 实例，不仅增加维护成本，还容易导致行为不一致、上下文割裂。更理想的做法是：一个 Agent 核心，通过统一的消息路由层，同时服务多个聊天平台。

OpenClaw 的插件体系与 MCP（Message Channel Protocol）扩展能力恰好为此提供了工程基础。本文记录一次将已有 OpenAI-compatible Agent 改造为跨平台 Bot 的过程，重点落在消息路由、上下文保持和常见踩坑点上。

## 问题拆解

要把一个 Agent 接到两个平台，需要解决三类问题：

1. **接入多样性**：Telegram 使用长轮询 / Webhook 获取更新，Discord 使用 Gateway 或 Webhook，消息载荷结构完全不同。  
2. **对话上下文**：同一用户在两个平台上的会话需要保持独立，也要避免跨平台串扰。  
3. **能力差异**：按钮、斜杠命令、附件、Markdown 格式在不同平台上的表现差异很大，Agent 回复必须适配对应渠道。

如果不抽象出一层统一的消息模型，核心 Agent 代码会被平台相关逻辑污染得很厉害。

## 做法：构建统一消息路由层

我在 OpenClaw 的基础上设计了一个轻量的 **Message Bus**，核心思路是：

- 每个平台对应一个 **Adapter**，负责将平台消息转换为标准消息（`StandardMessage`），并将 Agent 的回复转回平台可识别的格式。
- 一个 **Router** 根据用户 ID + 平台标识维护独立的会话上下文，并将标准消息投递给 Agent。
- Agent 自身只看到 `StandardMessage`，完全不知晓消息来源。

整体结构如图：

```
Telegram Bot → Telegram Adapter → 
                                 Router → Context Store → Agent Core
Discord Bot  → Discord Adapter  →
```

### 1. 统一消息协议（StandardMessage）

为了避免功能退化到“最小公分母”，我定义了一个结构体，同时保留平台相关的元数据：

```json
{
  "platform": "telegram" | "discord",
  "chat_id": "platform-specific-chat-id",
  "user_id": "global-user-id-or-platform-id",
  "text": "用户输入原文",
  "attachments": [{"type":"image","url":"..."}],
  "command": "/help 或 null",
  "metadata": { /* 平台特有信息，如按钮回调等 */ }
}
```

### 2. 平台适配器（Adapter）

每个 Adapter 实现两个主要方法：

- `inbound(platformEvent) -> StandardMessage`  
- `outbound(agentReply, platform) -> platformMessage`

以 Telegram Adapter 为例，`inbound` 负责解析 `Message` 或 `CallbackQuery`，提取文本、图片、用户名、chat_id 等，并标记 platform。  
`outbound` 遇到 Agent 返回的多图回答时，会根据 Telegram 的媒体组限制自动切分；超出 4096 字符的文本也会分批发送。

Discord Adapter 同理，但处理的是 `MessageCreate` 事件、`Interaction` 等。需要注意的是 Discord 支持嵌入消息（Embed）和按钮组件，如果 Agent 要使用这些高级特性，`outbound` 需要额外处理。

### 3. Router 与上下文存储

Router 维持一个内存级或持久化的会话上下文映射：

```
key: platform + ":" + chat_id
value: { conversationId, messageHistory[], lastActivity }
```

每次收到 `StandardMessage` 时，Router 从 key 中取出历史，合并当前消息，然后调用 Agent 的 `/chat` 接口。Agent 返回后，Router 将回复通过同一 key 对应的 Adapter 分发回去。

这里有个关键点：**避免跨平台共享上下文**。同一个用户可能在 Telegram 和 Discord 上同时提问，如果混合历史，Agent 会产生错误关联。所以 `user_id` 并不直接作为会话标识，而是一定要带平台前缀。

### 4. Agent 接入

我用的是兼容 OpenAI Chat API 的 Agent 后端，Router 将历史消息格式化为 `system`、`user`、`assistant` 角色数组后传入。Agent 配置简单的 system prompt，比如：

```
你是技术社区的助理。保持回答简洁、准确。不要提及平台名称，除非用户明确问。
```

这样 Agent 回复对各种渠道都是中性的，由 Adapter 负责格式化。

## 踩坑记录

- **长消息与速率限制**：Telegram 群组中如果连续发送多条长消息，容易触发 flood control。解决方案是在 Adapter 内实现串行发送，每两条之间延迟 200~400ms。Discord 也有全局速率限制，但通常 token-bucket 控制，需要根据 HTTP 头 `X-RateLimit-Remaining` 动态调整。
- **Markdown 差异**：Telegram 支持 `MarkdownV2`，需要对特殊字符转义；Discord 使用另一种 Markdown 子集，且支持角色/频道提及。Adapter 做 `outbound` 时应当做一层轻量渲染：例如将 Agent 输出的 `**粗体**` 分别转成 Telegram 的 `*粗体*` 和 Discord 的原生 Markdown。
- **附件问题**：Agent 可能返回图片 URL。Telegram 可以直接发送 URL 或 upload，但 Discord 需要将图片作为嵌入附件或使用 `attachments://` 协议。统一做法是 Agent 回复附件列表，Adapter 根据平台规则下载/重传或直接使用 URL。
- **命令处理**：斜杠命令在两个平台上都很常用。我不希望在 Agent 核心逻辑里判断命令，所以在 Adapter 层面将命令提取到 `StandardMessage.command` 字段。Router 可以基于命令快速路由，例如 `/reset` 清空上下文。
- **认证与安全**：确保每个平台的 Bot Token 隔离存储，Router 不要跨平台泄露数据。如果日志记录，要脱敏用户内容。

## 可复用建议

1. **从一端的完整接入开始**，比如先跑通 Telegram 的完整链路，再抽象出 Adapter 接口去接 Discord。不要一上来就设计“万能协议”。
2. **StandardMessage 不要过度泛化**，只提取你和 Agent 实际用到的字段。后期增加字段比过早设计容易。
3. **把平台特定逻辑关在 Adapter 内**，包括消息拆分、速率控制、格式转换。这样切换 Agent 后端或新增平台时影响面最小。
4. **会话上下文的存储**，初期可以用内存，但需要快速加持久化（Redis/SQLite），否则 Bot 重启丢历史会让用户很困惑。
5. **测试**建议用专门的测试群/频道，准备好图片、长文本、多语言、命令等各种用例，手动跑一遍远比写单测有效。

## 总结

这套方案并不复杂，核心在于用统一的消息协议和适配器模式将平台差异封闭起来。  
OpenClaw 的插件机制和 MCP 概念天然适配这类场景，只要划分清楚边界，一个 Agent 稳定地横跨 Telegram 与 Discord 完全可行，维护成本也比维护多个实例低很多。

实际运行一段时间后发现，最大的价值还不是省机器，而是用户在不同平台上能感受到同一个“助手人格”，这对社区体验提升非常明显。

---

