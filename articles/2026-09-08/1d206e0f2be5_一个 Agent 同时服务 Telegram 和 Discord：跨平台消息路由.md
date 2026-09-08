---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由的三个关键点
feedId: 36574
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

我们的社区在 Telegram 和 Discord 各有一个群组，之前是两个独立 bot、两份 prompt、两套脚本，知识库更新永远不同步，人格也越来越漂移。这次收敛成 OpenClaw 单 Agent：两个渠道共用同一份工作区（SOUL.md、技能、工具权限），运维成本直接减半。这篇只记录路由层面的实操，不聊 prompt 调优。

## 问题

多渠道接进来之后，真正要解决的是三件事：

1. **会话隔离**：同一用户可能同时出现在两个平台。上下文全合并会有隐私和串台问题，完全不打通又要在映射上花心思。
2. **格式适配**：Telegram 依赖 parse_mode，Discord 是自己的 markdown，且单条消息上限 2000 字符，长回复两边表现完全不同。
3. **群组行为**：Telegram 群有 privacy mode，Discord 有频道、线程、斜杠命令，触发逻辑不能共用一套假设。

## 做法

**第一步：渠道配置。** 在 `openclaw.json` 里同时声明 telegram 和 discord 两个 channel，各填 botToken。字段名以你本版本文档为准，结构大致是：

```json
{
  "channels": {
    "telegram": { "botToken": "***" },
    "discord":  { "botToken": "***" }
  }
}
```

**第二步：路由绑定。** 单 Agent 方案下，两个渠道都指向同一个 agent id。OpenClaw 默认按「渠道 + 会话」生成 session key，上下文天然隔离。如果某个 Discord 服务器需要独立人格，再通过 bindings 把该 guild 范围路由到另一个 agent，粒度到 guild/peer 级别。

**第三步：格式收敛。** 不要让 Agent 直接输出平台格式。我们约定 Agent 统一产出内部 markdown，发送层适配器负责转换和分片：Discord 侧超长自动切分，Telegram 侧处理实体转义。

**第四步：身份策略。** 平台 user id 只在各自平台内有意义，不要当全局主键。需要跨平台识别时单独维护映射表，权限校验以「渠道 + 平台 ID」二元组为准。

## 踩坑点

- Telegram bot 在群里默认收不到普通消息（privacy mode），要么在 BotFather 关掉，要么接受只有 /命令 和显式回复能触发。
- Discord 长回复直接 400 报错。分片别按固定字符数硬切，要按块边界切，避免代码块被腰斩。
- 同一条意图可能触发两次处理：Discord 的斜杠命令和普通消息、Telegram 的命令和正文，去重要放在路由层做。
- 两个渠道并发写同一 session 文件时，留意会话锁的行为；自定义插件里不要做额外的同步写。
- 本地开发别急着上 webhook。long polling 加 Discord gateway 已经够用，能省掉内网穿透这个变量。

## 可复用建议

1. 路由配置进 git，改绑定走 PR，出问题能回滚、能 diff。
2. 日志带渠道 tag。排障先看三行：消息从哪进来、路由到哪个 agent、session key 是什么。
3. 平台差异全部收敛在发送适配层，Agent 层保持平台无关。之后接第三个渠道，只是再加一个适配器。
4. 灰度顺序：先 DM 全量验证，再开放一个测试群，最后切生产群。

## 总结

跨平台消息路由拆开就是三件事：会话隔离定义上下文边界，格式适配抹平平台差异，权限分级控制暴露面。OpenClaw 已经把渠道适配和 session 管理做掉了，剩下的是把路由规则想清楚、写进配置、纳入版本管理。跑稳之后，新增渠道的边际成本会非常低。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/bdfdac9e88a58b73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/40a6521e8d76e245.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/674915c171fb790a.png)

