---
title: 一套大脑，两个入口：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 36140
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

我们的 Agent 核心早就稳定了：system prompt、MCP 工具、几个内部插件、会话记忆。问题出在入口——一开始只有 Telegram 群，后来团队又建了 Discord 服务器。最初的方案是复制一个 bot 实例各自挂插件，两周后两边 prompt 已经漂移，插件版本对不上，记忆也不互通。这才意识到：Agent 应该是服务，Telegram 和 Discord 只是两个 IO 适配器。

## 问题

把两个平台接到同一个 Agent 上，难点不在“同时在线”，而在消息模型差异：

- **格式**：Telegram 的 MarkdownV2 转义规则和 Discord 的 markdown 方言互不兼容；
- **限制**：单条消息长度 Telegram 4096、Discord 2000，分段逻辑不同；
- **线程**：Discord 有 thread / forum channel，Telegram 只有 reply 和话题群；
- **限流**：Discord 按频道约 5 条/5 秒，Telegram 全局 30 条/秒、单聊 1 条/秒；
- **身份**：同一个人在两个平台是两个 ID。

如果把这些 if/else 写进 Agent 主逻辑，很快就会失控。

## 做法

我们把它拆成四层：

1. **统一消息信封**。定义内部消息结构：`platform / chat_id / thread_id / user_id / content / attachments / reply_to`。两个渠道适配器只负责入站归一化，别的什么都不做。
2. **会话路由**。路由键用 `platform:chat_id:thread_id` 而不是 user_id——同一个用户在两个群里应当是两个上下文。Agent runtime 拿到信封后查 session，命中同一套核心。
3. **出站方言层**。Agent 产出统一格式（纯文本 + 代码块标记），发送前由各适配器转换：按代码块边界做长度裁剪、markdown 方言替换、附件统一走文件引用由适配器上传。
4. **限流与去重**。每个渠道独立令牌桶；流式回复不再逐 token 编辑，攒到句末或 2 秒窗口再 edit，避免触发 Discord 的编辑频率惩罚。入站侧用 Telegram 的 update_id 和 Discord 的 message id 做幂等去重。

## 踩坑点

- **Discord Message Content Intent 没开**：bot 在群里能收到消息事件但 content 全空，查了半天发现是开发者后台的一个开关。
- **Telegram bot 隐私模式**：默认在群里只收命令消息，普通提问全部丢失，需要去 BotFather 关闭。
- **用 user_id 做会话键**：早期版本同一用户跨群串上下文，A 项目的讨论混进了 B 群。
- **MarkdownV2 转义**：`_*[]()` 全要转义，最后放弃内联格式只保留代码块，稳定性立刻上来了。
- **双平台重复投递**：网络抖动重连时两边都会重发，没有幂等层之前 Agent 会把同一个问题回答两遍。

## 可复用建议

- Agent 核心里不允许出现平台名，所有差异锁死在适配器；
- 按渠道加 feature flag：长报告、图片渲染只在 Discord 开，Telegram 保持轻量；
- 日志统一带 `platform` 标签，排障时先按平台过滤再查链路；
- 渠道健康检查独立做，单渠道 API 故障时另一个照常工作，Agent 侧无感知。

## 总结

跨平台路由的本质不是“接两个 API”，而是把消息模型差异翻译层化。信封统一、路由键选对、出站方言分层、限流去重兜底——这四件事做完，将来接第三个平台（比如 Slack）只需要再写一个适配器，Agent 核心一行不改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/0df2cd3194340ef1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/c6acdf6fa601b035.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/84c1a06d886fcb1f.png)

