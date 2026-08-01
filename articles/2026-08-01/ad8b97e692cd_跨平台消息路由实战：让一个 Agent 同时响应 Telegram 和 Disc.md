---
title: 跨平台消息路由实战：让一个 Agent 同时响应 Telegram 和 Discord
feedId: 31204
source: 综合讨论
publishedAt: 2026-08-01
---

# 跨平台消息路由实战：让一个 Agent 同时响应 Telegram 和 Discord

## 背景：一个 Agent，多个入口

在基于 OpenClaw 的 Agent 实践中，最常见的部署模式是「一个 Channel + 一个 Agent」。Telegram Bot 接一个 Agent，Discord Bot 又接另一个。但当核心逻辑完全相同时，这种重复部署会造成维护负担：知识库、工具链、上下文策略需要多份同步。

更合理的方式是**单一 Agent 核心，同时服务于多个消息平台**。本文记录一次将同一个 Agent 同时接到 Telegram 和 Discord 的工程化过程，聚焦消息路由、格式适配和踩坑复盘。

## 问题拆解：不只是多开一个 Bot

表面看，只需启动两个 Channel Connector，各自连到同一个 Agent API。实际落地时，三个关键问题必须被显式处理：

1. **消息来源标识**：Agent 需要知道当前对话来自 Telegram 群组还是 Discord 频道，否则无法正确投递回复或维持上下文。
2. **格式鸿沟**：Telegram 使用 MarkdownV2 或 HTML，Discord 使用自己的 Markdown 子集，嵌入代码块、超链接的转义方式差异巨大，一个未转义字符就可能让消息发送失败。
3. **速率限制与资源争用**：两个平台的 API 限制机制不同，Telegram 同一 Bot 每秒最多 30 条消息，Discord 则基于全局滑动窗口，且有更复杂的 per-route 限制。若共享同一个网络出口，错误的突发重试会相互拖累。

## 实践做法：标准消息协议 + 平台适配器

### 1. 架构概览

采用 「Agent Core + Platform Adapters」 的分层设计：

```
Telegram User ─ Telegram Adapter ─┐
                                   ├─ Agent Core (HTTP/gRPC)
Discord User  ─ Discord Adapter ──┘
```

核心思想：Agent Core 只理解一套**标准消息格式**，完全不感知平台细节。每个平台适配器负责：

- 接收平台原生消息（Telegram Update / Discord Message Create event）
- 转换为标准格式，注入 `platform` 和 `chat_id` 元信息
- 调用 Agent Core 的统一接口
- 将 Agent 返回的标准回复转换为平台可接受的格式并发送

### 2. Agent Core 接口设计

一个最小可用的标准消息结构：

```json
{
  "platform": "telegram",
  "chat_id": "-1001234567890",
  "message_id": "12345",
  "text": "帮我查询本周日程",
  "timestamp": "2025-01-01T12:00:00Z",
  "reply_to": null,
  "attachments": []
}
```

Agent 处理完毕后返回：

```json
{
  "reply_text": "你本周有 3 条待办（Markdown 内容）",
  "actions": ["send_message"],
  "metadata": {
    "platform": "telegram",
    "chat_id": "-1001234567890"
  }
}
```

适配器根据 `metadata` 决定投递目标，并根据 `actions` 执行平台动作（发送文本、图片、按钮等）。

具体实现中，Agent Core 可以直接复用 OpenClaw 的 Agent 实例，只需将两个适配器的请求序列化后调用同一个 `handle_message` 函数。会话状态建议通过 `(platform, chat_id)` 的组合键隔离，避免 Telegram 的会话干扰 Discord 的对话。

### 3. 适配器要点

对于 Telegram 适配器，需要处理 Update 的长轮询或 Webhook，将 `Message` 中的实体（entities）展开提取纯文本，否则 MarkdownV2 的实体编码会污染传给 Agent 的原始文本。回复时再将 Agent 输出的 Markdown 转换成字符转义后的 MarkdownV2。

Discord 适配器则需监听 Gateway 的 `MESSAGE_CREATE` 事件，过滤掉自己的消息以防回声，并将提及（`<@user_id>`）转换为可读用户名或保留原格式。回复时检查内容长度，超过 2000 字符要分片发送或用文件发送。

## 踩坑记录

**坑 1：格式转义带来的不可见字符**

Telegram MarkdownV2 要求对 `. - ! # ( ) { }` 等特殊字符进行转义。若 Agent 返回了未转义的 `**重要**`，其中 `*` 不需要转义，但 `_` 需要。起初直接使用通用转义库，导致过度转义使格式失效。最终采用了「仅转义保留字符」的分平台函数，并通过单元测试覆盖常见用例。

**坑 2：速率限制连锁反应**

两个适配器最初使用同一个全局发送队列，当 Discord 因突发消息被全局限速（429）时，Telegram 的消息也全部被阻塞。修复方式是为每个平台分配独立的出站队列和退避策略，Discord 严格遵守 `retry_after` 头指示，Telegram 则用固定间隔控制。

**坑 3：大消息与附件处理**

Agent 在处理长文本或图片生成时，返回的 `reply_text` 可能包含本地文件路径。Telegram 可以通过 `InputFile` 直接上传，而 Discord 需要先上传到其 CDN 获得 URL 再发送 embed。如果不分平台处理，会导致 Discord 侧反复发送无附件的纯文本。最终在 actions 中增加了 `upload_image` 动作，由各自适配器实现上传逻辑。

## 可复用建议

1. **用适配器模式隔离变化**  
   核心 Agent 只依赖标准协议，增加新平台（如 Slack、Matrix）只需开发对应适配器。OpenClaw 本身已支持 Channel 抽象，可在此基础上封装。

2. **会话、限流、重试各自独立**  
   多个平台共享 Agent，但网络层（会话缓存、速率限制器、重试策略）应完全按平台拆分，避免相互影响。

3. **监控按平台维度打标**  
   Prometheus 指标应带上 `platform` 标签，便于观察各平台延迟、错误率、消息量。当某个平台出现异常时能快速定位。

4. **消息格式转换要写测试**  
   确保 `agent_md → telegram_md` 和 `agent_md → discord_md` 的转换逻辑有单测覆盖，尤其是边界字符和嵌套格式。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，本质上是一次「基础设施与业务逻辑分离」的工程实践。只要设计好标准消息协议和平台适配层，单个 Agent 就可以在多平台上保持一致的对话体验，同时避免重复开发。踩过的坑多集中在格式差异和速率控制，这两点值得在设计之初就明确隔离策略。

在 OpenClaw 生态内，完全可以基于现有的 Channel 机制实现这种跨平台消息路由，希望本文对正在实践多平台接入的同行有所参考。

---

