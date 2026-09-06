---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36330
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我们社区的用户一半在 Telegram，一半在 Discord。之前是两个 Bot 各挂一套脚本，问答逻辑重复维护，同一个问题两边答案经常不同步。这次重构的目标很明确：Agent 的"大脑"只有一份，Telegram 和 Discord 只是两种"嘴"。

## 问题

把两个官方客户端跑在同一个进程里并不难，难的是协议之外的东西：

- **格式方言**：Telegram 的 `MarkdownV2` 转义规则苛刻；Discord 是自己的 Markdown 子集，还有 embed。
- **长度与频控**：Telegram 单条 4096 字符、每 chat 约 1 msg/s；Discord 单条 2000、每通道约 5 条/5 秒。
- **会话身份**：同一个人在两个平台是两个 ID，session 怎么键控要提前想清楚。
- **附件语义**：Discord 附件 URL 现在带签名会过期；Telegram 的 file_id 还要另调一次 API 才能下载。

## 做法

整体分四层，核心原则是**适配器保持薄，路由和大脑共享**：

1. **统一消息模型**。定义内部 schema：`channel / user_id / text / attachments / reply_to`。入站时适配器只做一件事——把平台消息翻译成这个模型，不碰业务逻辑。
2. **两个薄适配器**。Telegram 走 long polling（推进 `offset` 即可），Discord 走 gateway（保留 `resume` 会话）。各自只负责收发和原始格式转换。
3. **共享运行时 + 会话键控**。Agent 核心和 MCP 工具只有一套，session key 用 `tg:<user_id>`、`dc:<user_id>` 区分。想做跨平台同人合并，单独建 identity mapping 表、按需 merge，别默认合并——隐私上要谨慎。
4. **出站渲染管线**。Agent 输出统一内部格式，出站前过 per-channel renderer：转义、按段落边界分段（维护一个代码块 fence 状态机，绝不从 ``` 中间切）。所有出站消息进同一个队列，队列里做通道级令牌桶限速。

部署上先单进程跑两个长连接；量大了再把适配器拆成独立进程加消息队列，接口不变。

## 踩坑点

- **分段切断代码块**是最高频事故，务必在渲染层带 fence 状态找分割点。
- **重连重放导致重复回复**：gateway 恢复后消息可能重投，用 `update_id` / `message_id` 做幂等去重。
- **MarkdownV2 转义**：`._*[]()~` 等一堆字符漏转义就整条 400，出站统一过 sanitizer。
- **Discord 频控是按通道 + 按 webhook 双维度**，别只按全局估算。
- **别长期引用平台附件链接**，收到就落自己的对象存储，签名 URL 过期后历史消息里的图全裂。

## 可复用建议

- 适配器薄、大脑厚，是这类架构唯一值得坚持的原则。
- 先用"最小公约 Markdown"（标题/加粗/代码块）覆盖 90% 场景，embed、按钮这类方言特性等真有需求再加。
- 日志和指标一定带 `channel` 标签，否则排障时分不清哪条腿瘸了。
- 别提前抽象"通用通道插件框架"，第三个通道真接入时再抽象也不迟。

## 总结

跨平台路由的工作量不在"连上两个 API"，而在归一化：消息模型归一、格式渲染归一、会话身份归一。这三件事做扎实，接第三个平台就只是一个新适配器的成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/3934e9a234527ab2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/38304c9b65da5705.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/21e8bec6a4227199.png)

