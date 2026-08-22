---
title: 一个 Agent 同时服务 Telegram 和 Discord：OpenClaw 跨平台消息路由实践
feedId: 34289
source: 综合讨论
publishedAt: 2026-08-23
---

# 一个 Agent 同时服务 Telegram 和 Discord：OpenClaw 跨平台消息路由实践

## 背景

如果团队一部分人在 Telegram 里做自动化提醒和脚本式交互，另一部分人在 Discord 里做社区讨论，很容易出现两个机器人各管一摊。后期维护两套 prompt、两套工具权限、两份记忆状态，改动一个逻辑要同步两边。本文记录基于 OpenClaw 做跨平台消息路由的实践：一个 Agent 核心，同时接入 Telegram 和 Discord，平台差异放在 adapter 层。

## 问题

直接接两个平台不难，难的是不让平台差异渗透到 Agent 逻辑：

- Telegram 与 Discord 的消息格式、回复引用、附件上传方式不同。
- 两边的 user id、channel id 命名空间互相独立，无法直接映射。
- 限流、重试、消息长度限制不同。
- 如果 agent 代码里到处写 `if platform == telegram`，后面加第三个平台会很痛苦。

## 做法/步骤

### 1. 准备 Bot

Telegram 用 BotFather 创建，拿到 token。Discord 在 Developer Portal 创建 Application -> Bot，开启 **Message Content Intent**，拿到 token 后邀请进服务器，授权 `Read Messages`、`Send Messages`、`Embed Links`、`Attach Files`。开发期 Telegram 用 webhook，Discord 用 gateway 或 webhook 均可。

### 2. 定义统一消息 envelope

在 OpenClaw 插件层定义一个平台无关的消息结构，所有 adapter 都转换成它：

```json
{
  "platform": "telegram",
  "chat_id": "12345",
  "user_id": "67890",
  "message_id": "999",
  "text": "查一下最近错误日志",
  "reply_to": null,
  "attachments": [],
  "raw": {}
}
```

Agent 核心只消费这个 envelope，不关心消息来自哪个平台。

### 3. 写两个 adapter

Telegram adapter 和 Discord adapter 各做三件事：接收平台事件、清洗并映射为 envelope、把 outbound envelope 转成平台格式发送。以 Discord 为例，收到 `message_create` 事件后，先过滤 bot 自己消息，再提取内容与附件，最后投递到 inbound queue。

一个最小配置片段：

```yaml
adapters:
  telegram:
    token: ${TG_BOT_TOKEN}
    allowed_chat_ids: ["-100xxx"]
  discord:
    token: ${DISCORD_BOT_TOKEN}
    guild_ids: ["xxx"]
routing:
  session_key: "platform:chat_id"
  outbound_queue:
    max_retries: 3
    base_backoff: 1s
```

### 4. Agent 与工具

OpenClaw 的 agent runtime 读取 inbound envelope，按 `platform:chat_id` 维护会话。工具调用走 MCP，例如 `log_query`、`db_read`、`http_fetch`。工具不依赖平台，因此一套工具能力可以同时给两个平台用。

### 5. 输出格式

Discord 优先用 Embed 承载结构化结果，回复时 @user；Telegram 用 HTML 而不是 MarkdownV2，减少转义出错。长内容按 2000 字符（Discord）、4096 字符（Telegram）分片发送。

## 踩坑点

1. **Discord Message Content Intent 不开启**：机器人能收到事件但 `content` 为空。需要在 Developer Portal 和客户端代码里都声明该 intent。
2. **Telegram MarkdownV2 转义过重**：`_*[]()~>#+-=|{}.!` 都要处理，稍不注意就发送失败。建议先转纯文本或 HTML。
3. **限流策略不同**：Discord 全局约 50 请求/秒，Telegram 全局约 30 消息/秒，单 chat 更严格。没有本地队列和指数退避，高峰时会丢消息。
4. **长消息限制**：Discord 普通消息 2000 字符，Embed description 4096；Telegram 文本 4096。需要统一分片函数，不能只按一个平台写死。
5. **附件转发陷阱**：Discord CDN 链接有过期签名，Telegram file_id 不能跨平台使用。需要先下载再上传到目标平台。
6. **忘记过滤自身消息**：如果 adapter 不过滤 bot 自己发送的消息，容易形成自我回复循环。
7. **权限模型混用**：不要直接拿平台 user id 当管理员。应用层要维护自己的一套用户/权限映射，否则 Discord role 和 Telegram 管理员关系会很乱。

## 可复用建议

- 把平台接入做成插件接口：`on_message`、`send_reply`、`upload_file`、`react`。接入新平台时只需实现一个新 adapter。
- Agent 不要直接发消息，统一投递 outbound envelope。发送失败进入重试队列，按平台配置限流和退避参数。
- 用 `(platform, chat_id)` 作为会话 key，避免跨平台串线。如需跨平台通知，显式维护用户绑定，不要隐式合并。
- 记录 `platform + message_id` 作为幂等键，避免 webhook 重试导致重复处理。
- 联调时用 webhook.site 或本地 tunnel 抓原始 payload，重点看附件字段结构和事件类型。

## 总结

跨平台消息路由的关键不是接几个平台，而是把平台差异隔离在 adapter 层，让 Agent 和工具保持平台无关。OpenClaw 的插件机制适合承担这个职责：adapter 负责 IO，agent 负责决策，MCP 负责扩展能力。这样做完后，同一个 Agent 可以在 Telegram 里接受命令，也能在 Discord 里参与讨论，不需要再维护两套逻辑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1b453e0d4f73861a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d5fe97741ced3fbb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/0b49668b5ac1443c.png)

