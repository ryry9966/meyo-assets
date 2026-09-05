---
title: 一个 Agent 同时服务 Telegram 和 Discord：双通道消息路由实践
feedId: 36250
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

最初接入 OpenClaw 时，多数人只挂一个 Telegram bot。用久了会发现使用者分散在两个生态：一部分人常驻 Telegram，另一部分社区协作在 Discord，同样的"问一下 Agent"要在两边重复操作。目标于是很自然：**同一个 Agent 实例、同一份记忆和工具，同时响应两个平台的消息。**

## 问题

有两条路：跑两个 OpenClaw 实例各挂一个通道，或者单实例多通道。前者的问题不是不能跑，而是状态割裂——两份 session、两份 memory、两份配置，改一次 prompt 要同步两处，长期维护成本很快失控。后者才是正解，但要想清楚三件事：**会话如何隔离、消息格式如何适配、如何防止回环。**

## 做法

**1. 双通道声明。** OpenClaw 的 gateway 天然支持多通道，在配置文件的 `channels` 下同时声明 `telegram` 和 `discord`，分别填入 BotFather 的 token 和 Discord Developer Portal 的 bot token，起一个 gateway 进程即可。两个通道的入站消息都会进入同一个 agent 核心，不需要额外路由代码。

**2. 会话路由。** OpenClaw 默认按「通道：会话ID」生成 session key，Telegram 私聊和 Discord 频道各自持有独立上下文，不会串味。如果想让某几个 Discord 频道走不同的 workspace 或人设，用 bindings 把频道绑定到指定 agent 即可，而不是再起一个实例。

**3. 行为差异配置。** 群聊务必设成"被提及才回复"：Discord 侧开 `requireMention`，Telegram 侧则要处理 privacy mode——群里的 bot 默认收不到普通消息，需要关闭隐私模式或设为管理员。长回复交给各通道 adapter 自行分片（Discord 单条 2000 字符、Telegram 4096），不用自己写切分逻辑。

## 踩坑点

- **Discord 不勾 Message Content Intent**：bot 能收到事件但 content 全为空，表现为"收到了但不说话"，这是最高频的坑。
- **Telegram 群 privacy mode 默认开启**：普通群消息进不来，很多人误判为路由故障，其实是收发层问题。
- **回环比双通道本身更容易翻车**：如果同时跑了一个 TG-Discord 消息桥，Agent 在两侧各回一次，桥再把消息搬回对面，就会形成乒乓循环。要么去掉桥，要么确保 adapter 忽略 bot 自身和其他 bot 的消息。
- **别手动拼 MarkdownV2 转义**，交给通道层处理，否则一次转义错误就会导致 Telegram 整条消息发送失败。
- 两个 token 不要提交进仓库，配置文件权限收到 600。

## 可复用建议

- 把通道当传输层，Agent 逻辑全部平台无关。prompt 里不要写"你是 Telegram bot"这类话，否则换平台就要重写。
- 单 gateway 多通道优于多实例，运维成本和状态一致性都更好。
- 日志里带上通道名和会话 ID，排障时能立刻定位是哪条链路的问题。
- 上线前在两个平台各测一遍私聊和群聊，共四种场景，尤其验证提及触发和附件（语音/图片）是否正常入站。

## 总结

这件事的大部分工作量是配置和策略，不是代码。OpenClaw 按"通道=传输、agent=大脑"分层，双通道接入几乎是声明式的；真正需要工程判断的是**会话隔离粒度、群聊触发策略、回环防护**这三点。想清楚了，一个 Agent 同时守两个平台是稳定可运维的，不需要任何黑魔法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/181b9c5e4fabc691.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/62376d17f3835869.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a317410c74c1bba4.png)

