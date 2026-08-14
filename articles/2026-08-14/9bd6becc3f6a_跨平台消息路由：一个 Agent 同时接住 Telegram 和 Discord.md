---
title: 跨平台消息路由：一个 Agent 同时接住 Telegram 和 Discord
feedId: 33108
source: 综合讨论
publishedAt: 2026-08-14
---

# 跨平台消息路由：一个 Agent 同时接住 Telegram 和 Discord

## 背景

很多自动化 Agent 一开始只接一个平台，通常是 Telegram 或 Discord。等用户群分散到两边后，容易演变成两套几乎相同的部署：两个 bot、两份配置、两个上下文，甚至两套命令。维护成本翻倍，行为还容易不一致。

如果要让一个 Agent 同时服务两个平台，关键不是“再开一个 bot”，而是把平台差异隔离在适配层，让核心 Agent 只面对统一的消息协议。

## 问题

直接在一个进程里同时连 Telegram 和 Discord，会遇到几类典型问题：

- 两个平台 API 差异大，消息结构不一致；
- 会话标识、用户 ID 体系不同，容易串号；
- 消息长度、格式、限流规则不同；
- 编辑、删除、按钮交互等事件模型不统一；
- 同一条 agent 回复需要按目标平台做格式转换和分段。

如果这些逻辑散落在业务代码里，后面每加一个平台都会更乱。

## 做法

下面是在 OpenClaw 中落地的一套轻量路由步骤，核心是“统一 envelope + 平台 adapter”。

### 1. 定义统一消息结构

所有平台进入的消息先被规范成同一种结构，例如：

```json
{
  "platform": "telegram",
  "chat_id": "-100123456789",
  "message_id": "456",
  "sender_id": "789",
  "text": "帮我查一下服务状态",
  "timestamp": 1710000000,
  "reply_to": null
}
```

核心 Agent 只消费这个结构，不关心它到底来自 Telegram 还是 Discord。回复也使用统一结构，由 adapter 负责真正发送。

### 2. 配置两个 connector

在 OpenClaw 中分别接入 Telegram bot token 和 Discord bot token。Telegram 可以用 polling 或 webhook；Discord 建议用 gateway 并开启 Message Content Intent，否则普通消息内容可能收不到。

每个 connector 的职责是：

- 接收平台事件；
- 转成统一的 envelope；
- 调用统一的 handler；
- 把 handler 的结果发送回原平台。

### 3. 会话键设计

避免跨平台串号，会话键必须包含平台维度：

```text
session:{platform}:{chat_id}
```

例如 `session:telegram:-100123456789` 和 `session:discord:987654321`。不要只用 chat_id，因为 Telegram 和 Discord 的 ID 体系完全不同。

### 4. 回复路由

handler 返回后，根据 envelope 里的 `platform` 字段决定走哪个 adapter。每个 adapter 负责：

- 内容格式转换；
- 消息分段；
- 限流控制；
- 发送结果回传。

## 踩坑点

### 消息长度

Telegram 普通文本上限是 4096 字符，Discord 普通消息上限是 2000 字符。同一个 agent 回复在 Telegram 能发出去，在 Discord 可能被拒。建议 adapter 层统一做分段，Discord 按 1800~1900 字符切分，避免边界问题。

### 格式不一致

Telegram 常用 MarkdownV2 或 HTML，Discord 只支持部分 Markdown 子集，并且嵌入、代码块行为不太一样。不要把 Telegram 的 MarkdownV2 直接发给 Discord，尤其要注意 `*`、`_`、`[` 这些字符需要按平台转义。

### 限流

Discord 对同一 channel 的发送限制大约是 5 条/5 秒，Telegram 全局限制较宽松，但单 chat 也有节流。建议每个 adapter 内置一个简单的令牌桶，不同平台独立配置，不要共用一个全局限流器。

### 编辑与删除事件

Telegram 会有 `edited_message`，Discord 会有 `message_update`。如果 agent 对每个事件都处理一次，就容易对同一条消息重复回复。初期可以只处理普通文本消息，明确忽略编辑和删除事件，等业务稳定后再按需支持。

### 命令冲突

两个平台都有 `\start` 之类的命令，但语义可能不同。命令表最好带上平台维度，例如：

```text
telegram:/status
discord:/status
```

不要设计成全局唯一的命令名，否则后续扩展会很痛苦。

## 可复用建议

- **适配器接口最小化**：每个平台 adapter 只需要实现 `normalize_input`、`send_message`、`split_text`、`check_rate_limit` 这几个方法即可。
- **消息失败要幂等重试**：发送失败可以重试，但要根据 message_id 去重，不能因为平台回调重试导致重复回复。
- **统一结构化日志**：日志里固定带 `platform`、`chat_id`、`message_id`，排查跨平台问题时效率高很多。
- **先只读灰度**：新接平台时，可以先让 agent 只接收并打印消息，观察该平台的事件类型和边界情况，再开放自动回复。
- **不要尝试自动跨平台识别同一用户**：除非用户主动绑定，否则 Telegram 用户和 Discord 用户应视为不同身份，避免权限混乱。

## 总结

一个 Agent 同时服务 Telegram 和 Discord，本质上是把“平台差异”封装进 adapter，让核心处理逻辑只依赖统一消息协议。这样 Agent 不需要知道消息来自哪里，后续再加 WhatsApp、Slack 等平台时，只需要新增 adapter，而不用改动核心逻辑。

这套模式不复杂，但能有效避免双平台部署带来的配置分裂和行为不一致。对于 OpenClaw 上的自动化实践来说，适合作为多平台接入的基础骨架。

---

