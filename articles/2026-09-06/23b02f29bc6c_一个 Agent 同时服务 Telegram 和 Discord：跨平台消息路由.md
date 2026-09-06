---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 36280
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我们最初只在 Telegram 上跑了一个基于 OpenClaw 的值班助手。后来团队另一批人的主战场在 Discord，希望"同一个脑子"直接接过去。最省事的做法是再起一个实例，但很快发现：两份会话状态、两套定时任务、双倍模型开销，而且同一个人在两边问同一个问题，会得到互相矛盾的答案。于是目标改成：**一个 Agent 核心，多条消息通道**。

## 问题拆解

真正要解决的不是"接上两个平台"，而是三件事：

1. **身份与会话**：同一个人在两边是两个不同的 ID，会话要不要合并；
2. **格式差异**：Discord 单条 2000 字符上限、支持 embed；Telegram 的 MarkdownV2 转义规则出了名挑剔；
3. **收发异步**：两个平台的到达节奏和限流策略完全不同，出站必须有队列。

## 做法

**第一步，通道做成适配器，核心只认统一信封。** 两个平台的插件只负责把消息归一化成同一结构：`platform、user_id、chat_id、message_id、text、attachments`。Agent 核心里不允许出现任何 `if platform == "telegram"`。

**第二步，定死会话路由键。** 我们用 `platform + chat_id` 起步：私聊按人、群聊按群，跨平台不合并。跨平台身份合并听起来美好，但要自己维护自然人映射表，还要处理"在 A 平台的私密讨论会不会泄到 B 平台"的信任问题，我们暂时放弃了。

**第三步，出站渲染与分片。** 长回复统一走渲染层：Discord 侧按 2000 字符切块、尽量不让代码块跨块；Telegram 侧统一用 HTML parse_mode，比 MarkdownV2 的转义地狱省心得多。

**第四步，防环、去重、限流。** 入站丢弃重复 `message_id` 和 bot 自身消息，防回声环；出站每个通道一条队列加令牌桶，Discord 触发 429 时按 Retry-After 退避。配置大致长这样：

```yaml
channels:
  telegram: { adapter: telegram, mode: long-polling }
  discord:  { adapter: discord, intents: [guild_messages, dm_messages] }
routing:
  session_key: "{platform}:{chat_id}:{user_scope}"
```

## 踩坑点

- Telegram bot 默认开 privacy mode，群里收不到普通消息，表现为"私聊正常、群里装死"。要么在 BotFather 关掉，要么把它设为群管理员。
- Discord 的 message content intent 要在开发者后台手动打开，忘了开只会收到空消息，日志里还看不出异常。
- 分块切进未闭合的代码块，后续消息整段格式错乱，分块点必须避开 ```。
- Agent 跑长任务时用户在另一平台追问，两条消息会并发命中同一会话，需要按会话加锁或排队，否则回复顺序会乱。

## 可复用建议

- 适配器只做归一化和渲染，平台方言永远不进 Agent 的系统提示词。
- 每条消息生成 trace_id，路由决策（进哪个会话、走哪个队列）单独打日志，排障时按 trace 串起来看。
- 先按"平台 + 会话"隔离上线，跑稳了再考虑跨平台身份合并，不要一步到位。
- 提供一个 dry-run 路由模式：只打印"这条消息会进哪个会话、渲染成什么"而不真正发送，适配层的回归测试全靠它。

## 总结

一个 Agent 服务多平台，难点不在"接上"，而在身份模型和格式边界。把通道压扁成适配器、把路由键想清楚、把渲染和限流隔离在核心之外，之后再加 Slack、飞书这类通道，基本只是再写一个归一化插件的事。欢迎在评论区交流你们的路由键设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/3da9d45b1a675ea8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/7470516aedbe838b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/cf5e87006f531345.png)

