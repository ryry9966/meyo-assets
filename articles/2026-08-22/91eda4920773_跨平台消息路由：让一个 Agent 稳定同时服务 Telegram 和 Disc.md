---
title: 跨平台消息路由：让一个 Agent 稳定同时服务 Telegram 和 Discord
feedId: 34236
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw/Agent 的实践里，很多场景需要一个 Agent 同时响应 Telegram 用户和 Discord 社区。如果每个平台各起一个实例，提示词、工具状态、记忆和配置会割裂，维护成本翻倍。更工程化的做法是：**一个 Agent 核心，多个平台适配器**。

## 问题

Telegram 与 Discord 的差异不仅是 API 不同，而是事件语义不同。Telegram 通过 webhook/getUpdates 推送 update，Discord 通过 gateway 派发事件；私聊、群聊、频道、线程、回复、编辑、按钮交互、附件等模型也不一致。如果让 Agent 直接消费原始事件，平台逻辑会侵入推理核心，后续加平台或改交互都很痛苦。

## 做法

### 1. 统一入站消息模型

定义一个最小 envelope，所有平台事件先转换成这个结构：

```json
{
  "platform": "telegram",
  "channel": "group",
  "chat_id": "-100123456",
  "user_id": "12345",
  "message_id": "tg:123",
  "text": "hello",
  "reply_to": null,
  "attachments": [],
  "raw": {}
}
```

`message_id` 使用平台前缀，避免跨平台冲突。`raw` 保留原始事件，便于排障和后续扩展。

### 2. 统一出站动作

核心 Agent 不要直接调用平台 API，而是产出统一动作：

```text
send_text(chat_id, text, reply_to?)
edit_text(chat_id, message_id, text)
react(chat_id, message_id, emoji)
send_media(chat_id, media_type, url, caption?)
```

每个平台 adapter 负责把这些动作翻译为对应 API 调用，并处理分段、格式转换、限流和重试。

### 3. 适配器与队列

Telegram adapter 接收 webhook，Discord adapter 接收 gateway 事件，二者只做协议转换，然后把 envelope 放入入站队列。Worker 串行消费，保证同一个 `platform:chat_id` 的消息按顺序处理。Agent 产生的动作进入出站队列，由对应平台 adapter 发送。

在 OpenClaw/MCP 体系里，发送能力可以包装成 MCP tool 或插件工具，让 Agent 通过工具调用发送消息，而不是依赖平台 SDK。这样 Agent 核心保持平台无关。

### 4. 基本流程

```text
Telegram webhook / Discord gateway
        ↓
平台 adapter 标准化
        ↓
入站队列
        ↓
Worker + Agent
        ↓
出站动作队列
        ↓
平台 adapter 发送
```

## 踩坑点

- **Discord 交互 3 秒限制**：斜杠命令或按钮必须快速 ACK。若 Agent 处理时间较长，先在 adapter 层做 deferred response，再异步更新。
- **长消息分段**：Telegram 单条上限 4096 字符，Discord 是 2000。分段逻辑应放在 adapter，并且尽量避免截断代码块或 URL。
- **限流差异**：Telegram 群聊建议控制在 20 msg/min 左右更稳；Discord 常见是每 route 5 次/5 秒。出站 adapter 应配置独立 token bucket。
- **重复投递**：Telegram webhook 重试、Discord 重连都可能产生重复事件。用 `platform:message_id` 做幂等判断。
- **Markdown 转义**：Telegram MarkdownV2 转义非常容易出错，建议统一使用 HTML parse mode 或纯文本，由 adapter 转换。
- **用户 ID 映射**：不要默认同一个人在两个平台是同一个人，除非有显式绑定。跨平台去重和身份合并要谨慎。

## 可复用建议

1. **平台逻辑限制在 adapter 内**：核心 Agent 只处理统一 envelope 和统一动作。
2. **先持久化，再处理**：入站事件先落库或写入队列，避免重启或异常时丢失。
3. **结构化日志**：日志字段至少包含 `platform`、`channel`、`chat_id`、`message_id`、`action`，排障效率会高很多。
4. **灰度发布**：先在一个频道或测试群跑通文本链路，再逐步开放按钮、附件、编辑等事件。
5. **配置黑白名单**：支持按 chat_id 或频道配置 Agent 触发权限，避免误响应。

## 总结

跨平台消息路由的核心不是“接两个 API”，而是建立一个稳定的统一消息模型，把平台差异收敛到适配器层。这样一个小型 Agent 可以同时服务 Telegram 和 Discord，共享同一套推理、工具和记忆状态，后续增加新平台也只需要实现新的 adapter，而不是重写核心逻辑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/516cc491a109f582.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/908950821f01ff26.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/242272cabc1a3962.png)

