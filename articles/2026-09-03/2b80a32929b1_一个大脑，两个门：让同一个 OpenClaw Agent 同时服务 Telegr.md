---
title: 一个大脑，两个门：让同一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 35924
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

不少人的 OpenClaw 最初只挂在 Telegram 上，后来团队或朋友迁到 Discord，于是顺手又部署了一套。结果就是：两份配置、两份记忆、两倍维护成本，同一个问题在两边问，Agent 给出两个不同口径的答案。这篇帖子记录把两个渠道收敛到单实例的做法——重点不在"怎么连上"，而在"怎么路由"。

## 问题

- **上下文分裂**：两个独立 Agent，记忆和工具配置不互通；
- **平台差异**：消息长度上限、Markdown 方言、群组触发机制全都不一样；
- **触发策略**：群聊里不是每条消息都该回，规则要按渠道分别声明；
- **身份映射**：同一人在两个平台 ID 不同，记忆和偏好要不要共享？

## 做法

**1. 单实例双渠道。** 一个 OpenClaw 进程同时配置 telegram 和 discord 两个 channel，各挂各的 token。Agent、系统提示词、MCP 工具只保留一份。

**2. 会话先隔离，后合并。** 默认按"渠道 + 会话"维度拆分 session。别急着打通，两边独立跑一两周确认稳定，再通过 bindings 把同一个人的两个平台 ID 指向同一 session。合并是优化项，不是前置条件。

**3. 触发规则按渠道声明。** Telegram：DM 直答，群聊仅在被 @ 或回复时触发；Discord：gateway 会收到全量消息，在路由层做 allowlist 过滤，@mention 或指令前缀才进 Agent。路由表配置化：

```yaml
channels:
  telegram:
    allowlist: ["<你的uid>"]
    trigger: ["dm", "mention"]
  discord:
    channel_allowlist: ["general", "ask-agent"]
    trigger: ["mention", "prefix:/ask"]
```

**4. 输出归一化。** system prompt 不出现任何平台名，Agent 只输出平台中立的轻量 Markdown；方言转换和长度分片全部下沉到各渠道适配层——Telegram 上限 4096，Discord 2000，切分时避开代码块内部。

**5. 身份映射。** 一张 SQLite 表存 `platform:uid → person`，记忆和偏好挂在 person 上而不是平台 ID 上，换平台不掉上下文。

## 踩坑

- **Discord 后台没开 Message Content Intent**，收到的消息 content 全是空。先查后台开关，再查代码——这条排查了我一小时。
- **Telegram 群组 privacy mode** 默认收不到普通消息，要么去 BotFather 关掉，要么只靠命令触发。
- **MarkdownV2 转义是黑洞**，别让模型输出它，适配层统一用纯文本或 HTML。
- **别拿 Agent 当桥**：把 A 平台消息转发到 B 后，Agent 会把转发内容当成新输入，形成回声循环。过滤 sender 为自身 bot 的消息，或给转发消息打标。
- **Discord 图片链接是签名 URL 会过期**，媒体消息在入队时就转存。
- 两平台同时来消息时，共享 session 会出现并发交错，**入队串行处理**比并行更稳。

## 可复用建议

- 复杂度放边缘：Agent 保持平台无知，方言、限速、分片全在适配层解决；
- 路由规则配置化，不写死在代码里；
- 日志带渠道前缀（`[tg]` / `[dc]`），排障效率完全不同；
- 工具与渠道解耦，一套 MCP 工具两边共享，不重复接。

## 总结

跨平台不是部署两份，而是"一个大脑、多个门"。会话策略先隔离后合并，触发规则按渠道声明，平台差异全部下沉到适配层。做完之后最直接的收益很朴素：维护一份配置，回答一个口径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/0a94a1fa37bac138.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/c7bed257c1d565a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/cb0deb90fa525db0.png)

