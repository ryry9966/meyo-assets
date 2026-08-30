---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 35473
source: 综合讨论
publishedAt: 2026-08-31
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

如果你正在用 OpenClaw 跑一个 Agent，想同时接 Telegram 和 Discord，最简单的做法是分别写两个 bot 入口，各自调用 Agent。但很快会遇到命令重复、状态分裂、日志难查的问题。更工程化的做法是把两个平台收敛到一条消息路由后面，让 Agent 只面向统一的 envelope 工作。

## 问题

两个平台的差异比看起来大：Telegram 是长轮询/getUpdates 或 webhook，Discord 需要 Gateway Intent 与 Interaction/Webhook 两套机制；消息长度、Markdown 语法、按钮回调、附件时效、限流规则都不同。如果不做抽象，平台逻辑会侵入 Agent 核心，后面每加一个平台都要再改一遍。

## 做法/步骤

1. 定义统一 envelope。建议至少包含：

```
{
  direction: "in" | "out",
  platform: "telegram" | "discord",
  chatId, userId, messageId?,
  type: "text" | "command" | "callback" | "media" | "edit",
  text?, attachments?[],
  ref?
}
```

平台 adapter 负责把 TG Update 或 Discord Message/Interaction 转成这个结构；Agent 输出再转回平台动作。

2. Adapter 接口保持最小。只需实现 `start()`、`stop()`、`send(envelope)`、`ack(callbackId)`、`edit(envelope)`。不要把平台 SDK 细节暴露给路由层。

3. 路由层维护会话映射。key 用 `platform:chatId`，不要尝试跨平台合并同一用户身份；TG 用户和 Discord 用户可能是同一人，但没有可靠依据前不要合并。会话状态放 Redis 或 SQLite，TTL 按需设置。

4. Agent 核心只处理 envelope。在 OpenClaw 里可以做成一个 router plugin，或通过 MCP 暴露 `route_message` 工具，让 Agent 输入/输出都是标准结构。这样后续加 WhatsApp/Slack 只需要新 adapter。

5. 输出侧做平台适配。Discord 单条消息 2000 字符，TG 4096；按 block 切分，避免在代码块中间断开。Discord 的 Markdown 不支持 TG MarkdownV2 的一些转义，统一先做 sanitize 再发送。附件建议转存对象存储，Discord CDN URL 有时效，TG 文件也有大小限制。

## 踩坑点

- **按钮回调必须立刻 ACK**。Discord Interaction 如果 3 秒内不响应，客户端会显示失败；TG callback_query 不调用 answerCallbackQuery，按钮会一直转圈。先 ACK 再异步处理，用 deferred 模式。
- **不要忽略 429**。Discord 全局和 route 限流很严格，TG 也有 flood control。发送侧要接指数退避，且退避期间不要重试同一条消息造成风暴。
- **编辑/删除是增强项，不是核心**。TG 支持编辑消息，Discord 也支持但权限和 webhook 行为不同。先实现“只发不回改”，避免为了全同步把主流程搞复杂。
- **日志要带 correlation id**。一次用户输入可能产生多次 Agent 输出和平台调用，用 `platform:chatId:messageId` 或随机 id 贯穿，排查时能看到完整链路。
- **Markdown 转义顺序**。先做文本切分，再做平台语法转义；反过来容易出现切分后转义失效。

## 可复用建议

- 先支持 text/command/callback 三类消息，跑通一周再加 media/edit。
- Adapter 层做 dry-run mode，只打印转换结果不真正发送，便于本地对拍。
- 用一个 `PlatformCapabilities` 字段声明平台支持什么（edit/media/callback），路由层据此降级。
- 对长输出用“分片器 + 脚标”或拆成多条，别让 Agent 自己拼 markdown 分片。
- 测试用真实账号建一个私聊 TG bot 和一个测试 Discord server，不要只靠 mock；平台真实行为经常和文档不一致。

## 总结

一个 Agent 同时服务 Telegram 和 Discord，核心不是“同时在线”，而是把平台差异关进 adapter，让路由层和 Agent 只跟统一 envelope 打交道。不要试图做完美抽象，先覆盖 80% 的文本和回调场景，后续按需扩展媒体与编辑能力。这样代码更稳定，排查也更清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9c7b91bf71f2091c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/03c9a26db8f7042e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e83d7815084ae027.png)

