---
title: 一套 Agent 核心，同时接住 Telegram 和 Discord：跨平台消息路由实践
feedId: 35976
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

我们的使用场景分两边：Telegram 群面向最终用户答疑，Discord 服务器面向开发者和内测反馈。早期是两个独立 bot，各自的 prompt、工具配置、记忆各一份，两周后行为就开始漂移——同一个问题两边答案不一致，排查起来很痛苦。于是做了合并：一个 Agent 核心，两个渠道只当"前端"。

## 问题

真正的难点不在"多连一个 API"，而在三件事不统一：

1. **消息模型**：Markdown 方言不同、长度限制不同（Discord 单条 2000 字符）、按钮和引用的表现完全不一样；
2. **身份与会话**：两边的 user id 毫无关联，私聊和群聊的上下文边界也不同；
3. **传输与配额**：Telegram 走 long polling/webhook 加全局限速，Discord 走 gateway websocket 加按频道限速，重连语义也各不相同。

## 做法

架构上分三层：薄的渠道适配器（adapter）→ 路由层 → 平台无关的 Agent 核心。MCP 工具和技能只注册一次，所有渠道共享。

1. **定义内部消息 schema**：`platform / channel_id / user_id / session_key / reply_to / media[]`。Agent 核心只认这个结构，不出现任何平台 SDK 的类型；
2. **统一 session key**：规则是 `platform:scope:id`。Discord 必须区分 DM 和 guild 频道，否则私聊上下文会漏进群聊；key 存 Redis，配 TTL；
3. **格式转换全放 adapter**：Agent 输出中性 Markdown，Telegram 侧统一转 HTML parse_mode，Discord 侧负责超长切分和代码块保护；
4. **异步回复带路由**：Agent 推理可能耗时数秒，回复消息必须携带 session_key，路由层据此投递回原渠道，并按 session 串行化，防止乱序；
5. **幂等与限速**：Telegram 用 update_id 去重，Discord 对 gateway 重连重放按 message id 去重；对外发送统一走带 Retry-After 退避的发送队列。

## 踩坑点

- **MarkdownV2 转义**：需要转义的字符集很大，漏一个整条消息 400。切到 HTML parse_mode 后此类报错归零，建议直接放弃 MarkdownV2；
- **代码块硬切分**：按 2000 字符直接切会把 ``` 切断，渲染错乱。切分前先解析块结构，块内不拆；
- **session 混用**：最初 DM 和服务器频道共用一个 session，用户在群里看到别人私聊话题的"尾巴"，上下文污染非常难看；
- **附件链接过期**：Discord 的签名 URL 有时效，跨渠道转发媒体必须先落盘再重新上传，不能只传 URL；
- **故障隔离**：一个渠道触发限速风暴时，不能阻塞另一个渠道的投递。adapter 独立进程、独立队列，配熔断。

## 可复用建议

- 依赖方向保持单向：adapter → core，核心代码永远不 import 平台 SDK；
- 内部 schema 带版本号，adapter 升级不破坏历史会话；
- 全链路 trace id，从入站消息到最终回复可检索；
- adapter 用录制的 fixture 回放做测试，不依赖真实 token；
- 渠道级 feature flag，新渠道灰度上线，出问题单渠道回滚。

## 总结

做完这轮改造最大的感受是：跨平台的价值不在"多接一个渠道"，而在于逼着你把 Agent 和渠道彻底解耦。路由层的核心工作归结为三件事——身份归一、格式归一、投递有序。这三件事立住了，之后接第三个渠道（Slack、邮件、Matrix）基本只是再写一个薄 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/e3954ff92b74f10a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c6fda51015e792f2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/b8077663f15821ad.png)

