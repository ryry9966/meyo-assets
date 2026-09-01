---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 35653
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

如果你已经在 OpenClaw 里把 MCP 工具链接到 Agent，下一步通常不是再加一个 Agent，而是让同一个 Agent 暴露到多个聊天入口。Telegram 和 Discord 是两个高频渠道，但直接写两套 bot 逻辑，会导致状态、工具调用和会话记忆割裂。

## 问题

同一个 Agent 同时服务两个平台，难点不在“能收到消息”，而在：

- 消息结构不同：TG 有 chat_id/message_id/reply_to_message，DC 有 channel_id/guild_id/interaction。
- 展示能力不同：TG 消息上限 4096，DC 2000；TG MarkdownV2 转义严格，DC 有 embed/mention。
- 限流与重试不同：双方都有 429，但 Discord slash command 要求 3 秒内 ACK。
- 会话身份不同：同一个自然人在 TG 和 DC 可能是两个身份，不能默认合并。

## 做法/步骤

我把这一层叫 transport，不碰 Agent 的工具调用和模型。Agent 仍通过 MCP 调工具，transport 只负责把异构消息归一化后交给 session manager。

1. 定义统一 Envelope：

```go
type Envelope struct {
    Platform      string
    ChatID        string
    ThreadID      string
    UserID        string
    Body          string
    ReplyTo       string
    Attachments   []Attachment
    IdempotencyKey string
    Timestamp     time.Time
}
```

TG 的 chat_id 映射为 ChatID；DC 的 channel_id 映射为 ChatID，thread_id 仅论坛频道使用。

2. 平台 adapter 只实现两个接口：`Receive()` 和 `Send(Envelope, RenderResult) error`。TG 用 webhook 或 getUpdates，DC 用 gateway。adapter 负责把平台事件转成 Envelope，并生成幂等键。

3. 路由 key 建议按 `platform:chat_id:thread_id?` 生成，不要按 user 全局合并。这样 TG 群和 DC 频道互不污染，同一个 Agent 在两边独立上下文。

4. 出站渲染需要区分目标平台。TG 优先用 HTML parse_mode 或纯文本，DC 长文用分片。分片建议按代码块和段落边界切，不硬截断。

5. 限流与失败处理：每平台单独出站队列，按 chat_id 限并发，429 时退避。TG 的 flood control 按 chat 生效，DC 的 429 通常带 retry_after。

可参考的最小配置：

```yaml
platforms:
  telegram:
    transport: webhook
  discord:
    transport: gateway
routing:
  key: "{platform}:{chat_id}:{thread_id?}"
queue:
  per_chat_concurrency: 2
  retry_intervals: [1s, 5s, 30s]
```

## 踩坑点

- **Discord slash command 会超时**：Agent 处理超过 3 秒时，必须先用 `defer` 或 `type: 5` ACK 后异步处理。不要阻塞 interaction 回调。
- **TG MarkdownV2 转义**：用户输入 `_`、`*`、`[` 等很容易导致消息发送失败。建议输出层统一转义，或改用 HTML/纯文本。
- **Webhook 重复投递**：两边 webhook 都可能重发。用 `IdempotencyKey` 建一张小表或带 TTL 的去重集合，不要只依赖内存。
- **编辑/删除事件**：TG 的 edit_message 和 DC 的 message_update 如果不需要回复，尽早丢弃，避免 Agent 把编辑当作新问题。
- **群聊噪音**：TG 群需要 BotFather 关闭 privacy mode 才能收到非命令消息；DC 频道需要相应 intents。上线前先确认事件范围。

## 可复用建议

- adapter 接口保持小，platform 特定逻辑不要漏到 Agent。
- 出站队列每平台独立，按 chat 串行，避免乱序。
- 日志带 platform、chat_id、event_id、latency、error_code。
- 健康检查区分 `/livez` 与 `/readyz`：进程活着不代表 TG/DC 连接正常。
- MCP 工具不必感知平台；如果工具输出需要展示，把它当成 render result 在 adapter 层适配，不要污染工具返回。

## 总结

单 Agent 多平台的关键是 transport 与 session 分离。不要为了省事把 TG 和 DC 的逻辑写进 Agent prompt，也不要在 adapter 里直接调用工具。统一 Envelope、平台 adapter、幂等键和按 chat 的出站队列，基本可以稳定支撑两个平台。后续加 Slack、Matrix 或企业微信，也只是多一个 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/81561b2b3751e5df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2b2c57f4691dfe44.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d1c909698b3c2b7a.png)

