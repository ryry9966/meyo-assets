---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34453
source: 综合讨论
publishedAt: 2026-08-24
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

当 Agent 开始进入真实聊天场景，常见需求是同一套推理逻辑同时服务 Telegram 和 Discord。两个平台的接入 SDK、消息格式、Markdown、速率限制、群聊语义都不同。如果直接在 Agent 核心里写 `if platform == "telegram"`，很快会变得不可维护。

本文记录一种轻量做法：把平台差异隔离在 connector 层，Agent 只面对统一消息信封。思路对 OpenClaw 这类 Agent runtime、MCP 工具链或插件化自动化场景同样适用。

## 问题

核心不是“能不能收到消息”，而是：

- 入站消息如何标准化？
- 会话和上下文如何隔离？
- 回复怎么回到正确的会话或线程？
- Markdown、消息长度、错误码等平台差异在哪里处理？

如果这些散落在业务逻辑里，接入第三个平台时成本会翻倍。

## 做法

### 1. 定义统一消息信封

所有 connector 先将平台消息转换成统一结构，例如：

```json
{
  "platform": "telegram",
  "chat_id": "-1001234567890",
  "thread_id": null,
  "sender_id": "987654321",
  "sender_name": "alice",
  "text": "帮我查一下构建状态",
  "message_id": "1111",
  "timestamp": 1710000000,
  "raw": {}
}
```

Discord 里 `chat_id` 可以用 `guild_id:channel_id`，`thread_id` 放论坛帖或线程 ID。私聊时 `chat_id` 直接用频道或用户 ID。

### 2. 会话键不要只用 chat_id

群聊里所有用户消息都会进入同一个 `chat_id`。如果 Agent 使用全局历史，就会把 A 和 B 的对话串掉。建议：

- 私聊：`platform:user_id`
- 群聊：`platform:chat_id:user_id` 或 `platform:chat_id:thread_id`

多个平台先不要合并同一自然人的身份，避免过早引入身份系统。等确认产品需求后再做身份归一。

### 3. Agent 层只处理标准化消息

Agent 的入口类似：

```text
handle(envelope) -> reply | action | null
```

输出不直接调用平台 SDK，而是返回统一动作，例如 `send_text`、`react`、`delete`。这样测试 Agent 时不需要真实平台环境。

### 4. 出站适配

出站 adapter 根据 `platform` 渲染消息：

- Telegram：使用 MarkdownV2 或 HTML，需要转义 `_*[]()~` 等字符。
- Discord：使用其 Markdown 子集，链接、代码块规则不一样。
- 长消息：Discord 单条 2000 字符，Telegram 单条 4096 字符；分段时优先在换行处切，避免切断代码块。

统一消息体可以使用逻辑标记，但每个平台要有独立渲染函数，不要只做一个全局替换。

### 5. 命令与触发

Telegram 天然用 `/cmd`，Discord 常用 slash command 或前缀 `!cmd`。建议 connector 层把命令解析成统一的 `intent`，并在配置中维护 allowlist。Discord slash command 注册有延迟，开发期可以同时保留前缀命令，方便调试。

### 6. 错误处理与重试

两个平台都有 429。Telegram 返回 `parameters.retry_after`，Discord 返回 `retry_after`。出站层要集中处理：

- 429 时退避重试，不阻塞其他消息。
- 其他 4xx 一般丢弃并记录，不要无限重试。
- 对用户可见的错误不要直接抛异常。

## 踩坑点

- **过滤 bot 自己**：Discord 和 Telegram 都会收到机器人自己的消息，需要在 connector 入口过滤 bot id 或 `is_bot`，否则可能形成回环。
- **Markdown 转义**：Telegram MarkdownV2 必须严格转义，Discord 不需要；同一段文本直接复用容易导致消息发送失败或格式错乱。
- **长代码块分段**：Discord 2000 字符限制很严格，分段时如果切断 ``` 代码块，剩余消息会丢失格式。最好按代码块整体搬运；无法搬运则先关闭代码块、拆分、再重新打开。
- **群聊响应策略**：在群聊中不要对每条消息都回复，应该只在被 @ 或命中命令时响应，否则用户体验差且容易触发平台限流。
- **消息 ID 去重**：connector 重连或 webhook 重试可能重复投递，用 `platform:message_id` 做幂等。

## 可复用建议

1. **Connector 独立成服务**：每个平台一个 connector 进程或容器，通过队列或 HTTP 把统一信封发给 Agent runtime。
2. **入站先持久化**：收到消息先落库，再处理；至少保证 at-least-once，配合幂等去重。
3. **平台配置外置**：token、允许的命令、群聊白名单、是否响应非 mention 消息，都放配置，不要在代码里硬编码。
4. **模板分平台**：出站消息模板按平台分目录，Agent 只给语义，渲染交给 adapter。
5. **可观测**：记录 `platform`、`chat_id`、`latency_ms`、`error_code`，出现问题先判断是 connector 层还是 Agent 层。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，不需要在 Agent 核心里写平台分支。关键是统一消息信封、明确的会话键、出站适配和集中错误处理。平台差异应被隔离在 connector 层，Agent 只处理业务意图。这套结构扩展到 Slack、Matrix 或其他 IM 时，新接入成本主要是写一个 connector，而不是改业务逻辑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/794fc6af6df2acf1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/3e1ecc3f6a6aa6db.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/e5c3377a1ad7901a.png)

