---
title: 跨平台消息路由：让一个 Agent 同时接住 Telegram 和 Discord
feedId: 34043
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw / Agent / MCP 的实践里，很多人习惯先把 Agent 接到单一 IM，比如 Telegram。但小团队内测或社区使用时，很自然会出现“Telegram 一批人、Discord 一批人”的情况。与其维护两个 Agent 实例、两套 prompt 和两份状态，不如把平台接入做成可替换适配层，让一个核心 Agent 处理两边消息。

## 问题

直接分别接入两个平台 SDK，最常见的问题是平台差异会渗入 Agent 逻辑：

- Telegram 用 MarkdownV2 / HTML，Discord 用 Markdown 子集，且单条消息限制 2000 字符。
- Telegram webhook 失败会重试，Discord Interaction 要求 3 秒内响应。
- 两边用户 ID、会话 ID、媒体 CDN 策略都不同。

最终代码很容易变成大量 if-else 堆叠，加第三个平台时基本不可维护。

## 做法

### 1. 抽象统一消息 envelope

让 Agent 只处理标准化结构，不感知平台。入站消息可以定义为：

```json
{
  "platform": "telegram",
  "chat_id": "123",
  "user_id": "456",
  "message_id": "789",
  "text": "hi",
  "media": [],
  "reply_to": null,
  "ts": 1710000000
}
```

出站也类似，只需 `platform`、`chat_id`、`text`、`reply_to_message_id` 等字段。核心 Agent 只面对这个结构。

### 2. 平台适配器

- **Telegram**：使用 `setWebhook` 接收 update，从 `update.message` 或 `update.edited_message` 提取内容。发送侧根据内部标记选择 `parse_mode`，或直接发纯文本。
- **Discord**：简单方案是用 Gateway 长连接监听 `MESSAGE_CREATE`。slash command 建议先不要碰，因为 Interaction 的 3 秒超时会引入额外复杂度。初版只处理普通频道消息和 DM 即可。

如果使用 OpenClaw，可以把适配层做成 MCP server 或独立插件，通过统一 inbox / outbox 接口暴露给 Agent。

### 3. 核心 Agent 接入

适配器把信封丢到队列，核心消费队列。处理前先发送 typing indicator：

- Telegram：`sendChatAction`
- Discord：`typing` 状态

长时间工具调用要避免让用户以为是 bot 死掉，最好有中间状态或 deferred update。

### 4. 出站渲染

内部尽量保留纯文本或受控标记。Telegram 用 MarkdownV2 时，需要转义 `_*[]()~`#+-=|{}.!`；Discord 只支持少量 Markdown。建议先输出纯文本，需要强调再按平台单独处理。超过 2000 字符时统一切片，优先保证 Discord 不报错。

### 5. 幂等

平台 webhook 重试会重复投递。以 `platform + message_id` 为唯一键做去重，TTL 24 小时足够。这个表可以放 Redis 或 SQLite，先检查再入队，避免重复触发工具调用。

## 踩坑点

- **Discord Interaction 3 秒超时**：如果直接把 slash command 接入 Agent，工具一调用就会超时。解决方式是先 ACK / defer，完成后用 `editReply` 或 followup 更新。初版禁用 slash command 可以绕开大量坑。
- **Telegram webhook 重试**：返回 non-2xx 会指数退避重试，重复消息可能造成重复执行。幂等键必须覆盖入站事件，不要只依赖 message_id。
- **用户身份不通用**：Telegram user_id 和 Discord user_id 没有映射关系。如果要跨平台会话，需要用户显式绑定，否则每个平台独立 session。
- **媒体消息**：Telegram 的 document / photo 和 Discord attachment CDN 链接都有时效。Agent 若只处理文本，要么把媒体转存到对象存储，要么明确返回“暂不支持附件”，否则上下文会丢失。
- **Discord 2000 字符限制**：长回复直接发送会报错。封装按段落边界切片，避免截断在代码块中间。
- **日志缺少平台维度**：排障时看不出消息来自哪个平台。每条日志至少带 `platform`、`chat_id`、`message_id`。

## 可复用建议

- 适配器做成 MCP server 或独立插件，让核心 Agent 只依赖统一接口。这样以后加 Slack / WhatsApp 不用改核心。
- 先只支持文本消息和基础命令，稳定后再加媒体、按钮、slash command。
- Discord intents 配置最小必要，例如 `GUILD_MESSAGES`、`MESSAGE_CONTENT`、`DIRECT_MESSAGES`。
- token、webhook secret 等用环境变量隔离，不要写进日志或 prompt。
- 预留 dry-run 模式：入站消息只记录、不执行工具，方便联调。
- 至少记录入站事件、出站结果、工具调用耗时、平台错误码。

## 总结

跨平台消息路由并不是复杂系统，关键在于把平台差异隔离在适配层。统一 envelope、幂等、平台渲染和超时处理是四个必须做对的点。做好这些，一个 Agent 同时服务 Telegram 和 Discord，不会比单平台多太多维护成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5493ca87086a50b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b7babcd760694de7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/db45a12ee2bfb90d.png)

