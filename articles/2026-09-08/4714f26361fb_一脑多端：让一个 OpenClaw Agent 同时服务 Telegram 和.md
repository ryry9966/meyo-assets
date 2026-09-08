---
title: 一脑多端：让一个 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 36607
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

场景很典型：个人助手跑在 Telegram 上服务私聊，另有一个 Discord 服务器作为小社区入口。最初偷懒开了两个 Agent 实例，各挂一个平台，两周后就绷不住了——两份记忆不同步，同一个问题在两边答案不一致，token 成本翻倍，改一处 prompt 得改两处。

结论很明确：**大脑必须只有一个，入口可以有很多个。**

## 问题

把两个平台接进同一个 Agent，难点不在"连上"，而在三件事：

1. **会话隔离**：Telegram DM 和 Discord 频道的上下文不能串；
2. **格式差异**：Agent 输出通用 markdown，但两边的渲染规则、长度上限（Telegram 4096 / Discord 2000）完全不同；
3. **能力差异**：回复、线程、reaction、附件，两边语义对不上。

## 做法

架构上保持"一个 Gateway 进程 + 两个 channel 适配器 + 一个 Agent 运行时"：

```jsonc
// openclaw.json（节选，字段名以你的版本为准）
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord":  { "token": "..." }
  },
  "session": { "scope": "per-channel" }
}
```

步骤：

1. **单实例部署**。一个进程同时加载两个渠道插件，共享 workspace 和技能（MCP 工具也只注册一次）。不要起两个实例再共享磁盘状态——会话锁会打架。
2. **会话键带渠道前缀**。用 `channel:scope:userId` 这类结构，如 `telegram:dm:10086`、`discord:guild:#general`。这是防串台的第一道闸。
3. **出口做格式化**。Agent 统一输出中性 markdown，Telegram 侧翻译成 HTML parse mode，Discord 侧保留原生 markdown；长度按平台分别分段。
4. **能力探测降级**。adapter 发送前检查平台能力：不支持线程就降级为普通回复，不支持富卡片就转纯文本。

## 踩坑点

- **409 抢消息**：同一 Telegram token 被两个进程同时 polling，getUpdates 直接互相踢。要么单实例，要么改 webhook。
- **上下文串台**：会话键漏了渠道前缀，Discord 用户在回复里看到了 Telegram 的私聊记忆——这个事故之后我们加了前缀强校验。
- **MarkdownV2 转义是地狱**：别硬扛，出站统一走 HTML parse mode。
- **Discord 全局限流更严**：长回复分段要留间隔，否则连续触发限速。
- **身份不合一**：同一个人在两个平台是两个 ID，记忆默认不互通；要共享就得显式做 identity mapping，否则对 Agent 来说就是"两个人"。

## 可复用建议

- **薄适配、厚核心**：adapter 只做三件事——收消息归一化、出站格式化、能力探测。业务逻辑别下沉到 adapter。
- **记忆与状态分离**：长期记忆和技能放共享 workspace，会话状态按会话键隔离。
- **渠道独立自愈**：两个渠道各自做健康检查和重连，一个平台挂了不影响另一个。
- **灰度顺序**：先跑稳一个平台再接第二个。接第二个平台时 agent 层几乎零改动——这本身就是架构正确的信号。

## 总结

跨平台消息路由的本质不是"多接几个 API"，而是：**归一化做在入口，格式化做在出口，隔离做在会话键里。** Agent 核心对此完全无感，才能保证以后接第三个、第四个平台时，依然是加配置，而不是改代码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/39ab2e89a3ae7d7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/76fd52b0740b156b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/eb671fb90e91c246.png)

