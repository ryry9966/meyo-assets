---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 32726
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

一次内部工具开发中，需求很明确：同一个问答 Agent，要同时接入团队用的 Telegram 群和社区用的 Discord 服务器。最初想法是起两个独立进程，分别对接两个 Bot，但很快发现维护两套代码、两套上下文管理成本不低，且后续增加新平台会继续膨胀。于是决定在 OpenClaw 中实现跨平台消息路由，让一个 Agent 实例同时服务多个消息通道。

OpenClaw 本身支持基于 MCP（Message Channel Protocol）的插件化接入，内置消息路由能力，但实际工程中需要处理的问题远比连接两个 webhook 要复杂。

## 核心问题

- **消息格式差异**：Telegram 消息是纯文本或 HTML/MarkdownV2，Discord 以 Embed 和 Markdown 为主。
- **平台特性不一致**：Telegram 有编辑消息、回调查询，Discord 有交互组件（按钮、下拉框）、线程。
- **回发目标映射**：Agent 回复时需要知道回哪条消息、哪个频道、哪个平台。
- **速率限制与重试策略**：Telegram 对频控敏感，Discord 对 API 调用更严格，需要统一但区别对待。
- **用户身份与会话维护**：同一用户可能在两个平台都发消息，如何区分？是否共享上下文？

## 做法与步骤

### 1. 定义平台无关的消息模型

在 Agent 内部，所有入站消息被转换成统一结构：

```
{
  “platform”: “telegram”,
  “channel_id”: “-100123456”,
  “user_id”: “123456”,
  “message_id”: “789”,
  “text”: “...”,         // 纯文本
  “raw”: {...},          // 原始 payload，供适配器使用
  “session_key”: “...”   // 由 platform+user_id+chat_id 生成的唯一标识
}
```

这样上层处理逻辑完全不需要知道平台细节。

### 2. 编写平台适配器

OpenClaw 支持通过插件注册 Channel Adapter。我为 Telegram 和 Discord 各写了一个，实现：

- **Receive**：将 webhook 内容转换为统一消息模型，写入 OpenClaw 的接收队列。
- **Send**：将 Agent 的回复从统一模型转回平台特定格式，并通过 Bot API 发送。
- **SendError/Retry**：处理发送失败，根据平台错误码决定重试策略。

Telegram 适配器中，长消息自动切分，代码块使用 `<pre>` 标签；Discord 适配器里，长消息转 Embed 的 `description` 字段，并限制总长度 4096 字符，超长则拆多个 Embed 或上传文件。

### 3. 配置路由规则

在 OpenClaw 中，消息路由可以基于 `platform` 和 `channel_id` 灵活配置。我们要求：

- 来自 Telegram 群的消息，正常回复到同一 Telegram 群。
- 来自 Discord 某指定频道，回复到该频道。
- 支持跨平台广播：管理员命令 `/announce` 可同时推送到两个平台。

通过路由配置表实现：

```yaml
routes:
  - matcher: {platform: telegram, chat_type: group}
    action: reply_to_source
  - matcher: {platform: discord, channel_id: "108..."}
    action: reply_to_source
  - matcher: {text: starts_with("/announce")}
    action: broadcast
    platforms: [telegram, discord]
    target: default_channels
```

OpenClaw 内部会根据 matcher 将消息转给 Agent 对应的 handler，并记录 source 用于回复。

### 4. 会话与上下文

因为用户可能在两个平台都用同一个问题，我们不希望上下文混淆。因此 `session_key` 生成规则中包含 `platform` 前缀，例如 `tg:123456:group_abc` 和 `dc:98765:channel_xyz`。Agent 的上下文管理按 `session_key` 隔离，但也允许通过用户 ID 绑定实现跨平台个人历史查询（需显式授权）。

## 踩坑记录

- **Telegram Markdown 转义不一致**：MarkdownV2 与 Discord Markdown 的差异巨大，统一用纯文本输出，格式标记由适配器转换，但若用户输入含 `*` 等特殊字符，易导致解析异常。最终决定 Agent 只输出纯文本，适配器对 Telegram 做特殊字符转义（用 `MarkdownV2.escape()`），Discord 则按 Markdown 渲染。
- **Discord 消息长度限制**：当 Agent 返回近 3000 字的详细答案时，直接发送会报错。解决方式是适配器自动拆分成多条消息，但还得维护消息发送顺序和编辑时的整体性。折中方案是超过 2000 字符时生成摘要，并附加一个指向知识库的链接，避免刷屏。
- **跨平台广播的状态同步**：/announce 命令需要在两个平台都发送成功后才会确认，其中一个失败即整体失败并重试。使用 OpenClaw 的 task pipeline 保证原子性，避免只发了一个平台。
- **速率限制处理**：Telegram 对同一 chat 的发送速率约 20 条/分钟，Discord 为 5 次/5 秒。适配器各自实现令牌桶，并在 429 错误时自动等 `retry_after` 毫秒，而不是简单的固定延迟。

## 可复用建议

1. **抽象适配器接口**：定义 `receive`、`send`、`validate` 等通用方法，新平台只需实现该接口，核心 Agent 逻辑零改动。
2. **使用中间件处理通用逻辑**：如幂等性、重试、消息去重等，放在平台无关的 middleware 层。
3. **配置驱动路由**：不要硬编码平台判断，用规则匹配增强灵活性。
4. **格式处理外包给适配器**：Agent 只管纯文本或结构化数据，由适配器负责平台化渲染。
5. **测试用 Stub 平台**：写一个 Console Adapter，可在本地终端模拟多平台发送，快速验证路由和格式。

## 总结

一个 Agent 服务多个平台的核心不在“连接”本身，而在工程上如何隔离平台差异、保证一致的消息路由与可靠投递。OpenClaw 的通道抽象加上 MCP 插件化让这件事变得可行，但实际投入生产时，务必为每个平台的怪癖写足适配代码。这个实践节省了我们至少 40% 的重复开发工作，且当需要接入 Slack 时，仅需两天就完成了新适配器与路由配置。

---

