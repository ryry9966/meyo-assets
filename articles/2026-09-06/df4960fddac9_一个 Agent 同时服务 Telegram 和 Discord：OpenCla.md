---
title: 一个 Agent 同时服务 Telegram 和 Discord：OpenClaw 跨平台消息路由实战
feedId: 36306
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我们的用户天然分成两拨：技术向的泡在 Discord，轻量用户习惯 Telegram。最初的做法是跑两个独立实例，各一套配置、一套记忆，结果是同一个助手在两边的"人格"和上下文完全对不上，工具配置（浏览器、脚本、知识库）也要双份维护。这次改造的目标很朴素：一个 agent 实例，两个通道收发，行为一致、会话隔离、维护单点。

## 问题

把两个 channel 的 token 填进配置只是起点。实测下来，真正的成本集中在这几处：

1. **格式方言**：同一份回复在 Discord 渲染正常，到 Telegram 变成裸星号，甚至因转义失败直接 400；
2. **会话键设计**：按通道隔离还是按人合并，直接影响上下文是否串扰；
3. **长度与分段**：Discord 单条 2000 字符，Telegram 4096，切分策略两边不同；
4. **可见性权限**：Discord 的 Message Content Intent、Telegram 群的 privacy mode，任何一个没开，bot 就是"半聋"状态。

## 做法

**第一步：双通道配置。** 精简后的结构大致是：

```jsonc
{
  "channels": {
    "telegram": { "botToken": "<tg-token>", "dmPolicy": "allowlist" },
    "discord":  { "botToken": "<dc-token>", "dmPolicy": "allowlist" }
  }
}
```

字段名以你手上版本的文档为准，关键是 `dmPolicy` 先收紧，灰度期只放行测试账号。

**第二步：确认会话键。** OpenClaw 默认按「通道 + 对端」派生 session key，即同一个人在两个平台是两个独立会话。我们评估后保留了这个默认：合并记忆听起来美好，但跨平台上下文一旦串了，排查成本远高于收益。确有合并需求就用显式 identity 映射，别依赖隐式行为。

**第三步：出站格式适配。** 约定一个"安全 Markdown 子集"：粗体、行内代码、无序列表、引用，仅此四样。出站层按目标通道转换：Telegram 走 HTML parse mode，避开 MarkdownV2 的转义地狱；Discord 侧注意代码块不要被分段切断。长度按各平台上限取小值再分段。

**第四步：群内触发策略。** 两通道统一为"群内必须 @提及 才响应，私聊直接响应"，避免 bot 在大群里抢话，也省 token。

**第五步：灰度验证。** 先在测试群把收发、媒体（图片、语音）、长回复、中断重试各过一遍，再放量。

## 踩坑点

- **Discord 空消息**：开发者后台没开 Message Content Intent，网关连上了但内容全为空，现象很像上游模型故障；
- **Telegram 群隐私模式**：默认只收 /命令 和 @ 消息，要么关 privacy mode，要么把 bot 设为管理员；
- **MarkdownV2 转义**：下划线、点号没转义就是 400，这是我们直接换 HTML mode 的原因；
- **分段切坏代码块**：按字符数硬切会把 fence 劈成两半，分段逻辑要感知代码块边界；
- **响应节奏差异**：Discord 用户习惯连发追问，回复队列必须按序处理，乱序会显得答非所问。

## 可复用建议

1. **系统提示保持平台中立**，平台差异写成一小段 per-channel 附加说明注入，别复制两份提示词；
2. **出站统一走安全子集 + 通道适配层**，将来新增通道只写一个适配器；
3. **allowlist 先于功能上线**，权限模型永远比功能先就位；
4. **日志保留完整 envelope**（通道、chat id、message id、时间戳），排障效率差一个量级。

## 总结

跨平台路由不是"多填一个 token"的事，真正的工作量在格式适配、会话键和权限模型这三件不起眼的事上。OpenClaw 的 channel 抽象把大部分差异挡在了配置层，剩下的坑踩一遍就够了。推荐路径：收紧权限 → 灰度测试群 → 验证媒体与长消息 → 再放量。一套 agent 服务多通道的收益是长期的：配置单点、记忆一致、工具复用，值得一次做对。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a519b2c644d036d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/ae5b5b3837835648.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/44ed2bc1cf8b5bb0.png)

