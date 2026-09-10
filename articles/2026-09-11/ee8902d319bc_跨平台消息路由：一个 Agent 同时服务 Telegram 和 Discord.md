---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36979
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

我们的 Agent 最初只挂在 Telegram 上服务运营群，后来发现协作讨论有一半跑在 Discord 里。最直接的做法是再写一个 Discord bot，但很快出现两个问题：两套 prompt 和工具配置各自漂移，改一处忘一处；排障时要在两份日志之间来回翻。所以这次重构的目标很明确：一个 Agent 核心，两个平台入口，配置一份。

## 问题

真正动手才发现两个平台的差异比想象大：

- **推送机制不同**：Telegram 走 webhook 或 polling，Discord 是 gateway websocket + 心跳重连；
- **消息模型不同**：Markdown 方言互不兼容，附件、回复引用的表示完全不一样，长度限制也不同（Telegram 4096，Discord 2000）；
- **限流策略不同**：Discord 按路由 bucket 限流，Telegram 有全局广播限制；
- **身份体系不互通**：user_id 无法对齐，会话归属需要自己定义。

## 做法

核心思路一句话：**adapter 负责归一化，核心层不感知平台**。

1. 定义统一消息结构 `IncomingMessage / OutgoingMessage`，字段包括 `platform`、`chat_id`、`user_id`、`text`、`attachments`、`reply_to`、`message_id`。
2. 每个平台一个 adapter，只做三件事：收消息转统一格式、发消息做格式转换和分段、维护连接（重连、心跳）。
3. 会话主键用 `platform:chat_id`，同一人在两个平台默认独立上下文，避免串味；确需打通时在路由表显式指向同一 session：

```yaml
routes:
  - platform: telegram
    chat_id: "-100123..."
    session: ops
  - platform: discord
    channel_id: "1122..."
    session: ops   # 复用同一会话，按需打通
```

4. Agent 内部统一输出标准 Markdown，adapter 各自降级转换，长度分段也在 adapter 里做。
5. 长任务统一 ack：收到消息先发 typing 状态或占位消息，完成后编辑原消息。两个平台各自实现 `ack() / finalize()` 接口。
6. 把「发送消息到指定会话」注册成 MCP 工具，Agent 就能主动跨会话推送，比如定时任务结果。

## 踩坑点

1. **Telegram webhook 和 getUpdates 不能共存**。本地调试开 polling 会抢走线上 webhook 的消息，表现为线上随机丢消息。调试用单独 token，或显式调 `deleteWebhook`。
2. **Discord 分段回复连发容易 429**。adapter 里加了串行发送队列 + 指数退避才稳。
3. **分段不能按字符切**，会把代码块拦腰切断。改成按段落和代码块边界切。
4. **平台重试导致重复投递**。用 `message_id` 做幂等去重，五分钟窗口内重复 ID 直接丢弃。
5. **群聊噪音**：Discord 服务器消息全量推送，必须做 mention/前缀触发过滤，否则 token 消耗爆炸。
6. **附件回传**：Agent 生成的图片外链在 Discord 里经常不出预览，最稳是 adapter 直接上传为平台原生附件。

## 可复用建议

- adapter 只做翻译，禁止写业务逻辑；核心代码不 import 任何平台 SDK。
- ack、typing、占位消息抽象成统一接口，新平台接入只需实现一个类。
- 日志统一打 `platform:chat_id:message_id`，跨平台能追完整链路。
- 先把单平台跑稳再抽接口，不要一上来设计「万能 adapter」。
- 上下文是否跨平台共享做成配置项，默认隔离，出问题好回退。

## 总结

跨平台消息路由的难点不在「多接一个 SDK」，而在消息归一化和会话边界这两件枯燥的事。adapter 做薄、核心做厚，限流和 ack 留在 adapter，会话和业务收敛到核心，之后接 Slack、飞书，本质上只是再写一个 adapter 的工作量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/69709177dae031f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/f2d015f1b75bc9ad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/4b60f71c42ba08d6.png)

