---
title: One Agent, Two Channels: 统一消息路由让同一个 Agent 同时服务 Telegram 与 Discord
feedId: 31094
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景

有独立开发者或者小团队在维护一个基于 LLM 的 Agent（助手），最初可能只挂在一个 Telegram 群／频道里，效果不错。随后团队想在 Discord 社区也提供同样的能力，最偷懒的做法是：起两份 Agent 实例，各自接入不同平台，互不干扰。这样虽然能工作，但维护两套几乎相同的配置、模型调用、工具执行逻辑，很快就变得痛苦——版本不一致、统计数据割裂、更新一个 skill 要改两个地方。

本文基于 OpenClaw 这类 Agent 运行框架实战经验，整理一种“单 Agent 多平台”的工程实践：同一个 Agent 核心，通过统一的消息路由层，同时服务 Telegram 和 Discord 两个平台，并保持会话隔离与上下文管理。重点不在于接入本身的 Hello World，而在于如何组织和抽象，避免随着平台增多而失控。

## 问题拆解

- **消息格式异构**：Telegram 的 `Message` 结构、实体解析、媒体类型，与 Discord 的 `Message`、`Interaction`、Embed 完全两套模型。
- **会话标识不同**：Telegram 用 `chat_id`（可能和 user 绑定的 private chat，或群组），Discord 是 `guild_id` + `channel_id`。用户身份一侧是 `user_id`（纯数字），另一侧是 snowflake。
- **回复约束不一致**：文本格式（MarkdownV2  vs  Markdown 子集／Embed）、消息长度上限（4096 vs 2000）、富媒体支持方式不同。
- **网络和协议差异**：Telegram 可用 long polling 或 webhook，Discord 用 WebSocket Gateway + REST。保持二者长连接稳定、错误恢复策略不一样。
- **并发与速率限制**：各自 API 的 rate limit 颗粒度不同，若一个平台的流量异常可能影响同一进程内的另一个平台。

## 统一消息路由的设计

核心思路：在 Agent 内核与平台适配层之间插入一层 **Message Router**，将不同平台的原始事件转化为统一的 `AgentContext`，处理完后再还原为平台特定的回复动作。

### 1. 统一消息模型

定义内部消息对象，字段尽量精简但覆盖必要元信息：

```ts
type AgentMessage = {
  id: string;
  platform: 'telegram' | 'discord';
  sessionKey: string;      // 跨平台唯一会话标识
  userId: string;
  displayName?: string;
  content: string;        // 已清洗的纯文本输入
  raw: Record<string, unknown>; // 保留原始 event object
};
```

`sessionKey` 的构造非常重要，直接决定对话历史存到哪里。我们使用 `platform:chatId` 作为 key（Telegram 就是 `tg:123456`，Discord 用 `disc:GuildID-ChannelID`），确保即使同一用户跨平台私聊也不会串线。

### 2. 平台适配器

为每个平台实现简单的 **Adapter** 接口：

- `parse(rawEvent) → AgentMessage[]`：解析 webhook／WebSocket 收到的原始事件，可能 1 个事件产生多条 Normalized Message（比如 Telegram channel post 和 callback query）。
- `formatReply(ctx, result) → PlatformReply`：把 Agent 输出转成对应平台的发送参数（Markdown 格式、Embed 字段、按钮等）。
- `send(reply) → Promise<void>`：实际调用平台 API 发送，内部处理速率限制、重试等。

OpenClaw 的 MCP 工具生态可以进一步简化这一步：将 Telegram Bot API 和 Discord REST 封装为 MCP 工具，Agent 在需要发送富媒体、执行平台特定操作时直接调用工具。但路由层仍然必须处理初始的消息投递和回复分发。

### 3. 会话隔离

利用 OpenClaw 的 session manager，以 `sessionKey` 存储对话上下文。这样可以做到：同一个用户在不同平台上和 Agent 对话是独立的、可追溯的。同时如果想让一些经验跨平台共享（例如用户告知偏好），可以在 profile 层合并，而不污染 session。

### 4. 路由编排

主循环负责：

1. 从 Telegram 和 Discord 的 listener（比如 Express webhook handler + Discord client event）收集事件，放入一个统一队列。
2. 队列消费者取到原始事件后，通过对应 Adapter 解析为 `AgentMessage`。
3. 携带 `AgentMessage` 调用 Agent 内核（触发思考/工具链）。
4. 得到 AgentResponse 后，使用同一 Adapter 格式化并发送回复。

关键点：消费者是平台无关的，Adapter 的选取由 `platform` 字段决定，因此核心逻辑中不出现平台分支。

## 踩坑记录

- **Telegram long polling 与 webhook 选型**：单进程多平台下，使用 webhook 更好，避免 poll loop 阻塞事件循环或与 Discord WebSocket 抢占资源。注意在开发环境无法接收公网 webhook 时，可用 `ngrok` 或类似工具，但要做好鉴权与 IP 白名单。
- **Discord Gateway Intents**：如果 Agent 需要读取消息内容（非命令），必须开启 `MESSAGE_CONTENT` intent，且 Bot 需要对应权限，否则任何 `messageCreate` 事件里 content 为空。这个坑让很多一开始只做 slash command 的 bot 在迁移时措手不及。
- **消息长度处理**：Telegram 上限 4096 字符，Discord 2000。当 Agent 输出超长时，必须在 Adapter 层做分段。实践中可以先尝试分段，如果仍有超过限制的，降级为文本文件。
- **速率限制**：Telegram 同类方法 ≈30 msg/s，Discord 有 per-route 限制。最好对每个平台实现独立的 ratelimiter，而不要用一个全局限流器，否则一个平台受限会导致另一个平台不必要的等待。
- **错误隔离**：任何平台的 dispatch error（如用户 block bot、网络超时）不应该导致另一平台的事件丢失。可以用 per-platform 的 try-catch 加上错误日志，并确保队列消费里的 `catch` 不会造成未处理拒绝。
- **身份映射**：如果在 Agent 内部需要关联用户在两个平台的身份，可维护一个简单的 profile service 通过验证码或共同 email 进行绑定，尽量不要直接用 LLM 去猜。

## 可复用建议

- **从第一天就设计 Message 抽象**：即使现在只有一个平台，也用 Normalized Model，接入第二个平台时只需新增 Adapter，不碰核心。
- **平台特定功能走内部指令**：例如 Discord 的 Embed 回复，Telegram 的 inline keyboard，不要强迫 Agent 输出 JSON 再解析，而是让 Agent 输出一种内部的标记（`<embed>...</embed>`），然后 Adapter 解析。这降低了 prompt 的复杂度。
- **环境变量管理**：`TELEGRAM_TOKEN`、`DISCORD_TOKEN`、`BOT_WEBHOOK_PATH` 等通过环境注入，避免硬编码，同时支持切不同 bot 账号调试。
- **可观测性**：对每个平台单独记录“解析失败次数”、“发送延迟 p99”、“限流触发次数”等，方便定位问题。

## 总结

同一个 Agent 同时服务 Telegram 和 Discord，技术上并不需要复杂的微服务拆分，关键在于把平台差异封装在适配层，保持消息路由的纯净。通过设计统一的 Message 模型、合理的 sessionKey 策略以及独立速率控制，可以确保 Agent 的行为一致、可维护、可扩展。当未来需要接入第三个平台（例如 Slack）时，只需再写一个 Adapter，核心逻辑零改动。

这种多平台路由模式非常适合已在使用 OpenClaw 搭建自动化助手的团队，它能让开发者的精力聚焦在 Agent 能力本身，而不是不断重复平台接入的脏活。

---

