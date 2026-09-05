---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36162
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

我们的 OpenClaw 实例最早只挂在 Telegram 上，服务内部值班群。后来一部分协作搬到了 Discord，社区答疑和长讨论都在那边。第一反应是再起一个实例，但很快发现不对：两个实例意味着两份记忆、两套配置、两组密钥，最麻烦的是同一个人在两边提问，Agent 认不出他是同一个人。

于是改成单实例多通道：一个 Agent 进程，同时挂 Telegram 和 Discord 两个 channel，中间加一层薄薄的路由。稳定跑了一个多月，把做法记录如下。

## 问题拆解

单实例双平台，本质上是四件事：

1. **接入**：两边协议不同（Telegram 长轮询/webhook，Discord gateway WebSocket），不能在 Agent 主体里各写一套；
2. **会话隔离**：Telegram 群 A 的上下文不能漏进 Discord 频道 B；
3. **身份映射**：两套 user id 体系，不映射的话记忆和权限对不上；
4. **输出适配**：Markdown 方言、长度限制、引用机制都不一样。

## 做法

架构上坚持「接入层薄、Agent 层厚」：

```
Telegram ─┐                     ┌─ Telegram
          ├─ 路由/适配层 ─ Agent ┤
Discord ──┘                     └─ Discord
```

1. **统一信封**。channel 插件只做两件事：收消息后归一化成统一信封（platform、chat_id、user_id、内容、引用），出站时按平台渲染。Agent 永远只面对一种消息结构。
2. **会话键设计**。session key 用 `platform:chat_id`（如 `tg:-100xxx` / `dc:channel_id`），天然隔离上下文。需要跨平台共享的部分走长期 memory，不要塞进 session。
3. **身份映射表**。维护一张 `tg_id <-> dc_id -> 内部身份` 的映射，人不多，手动维护即可。记忆查询和权限判断都走内部身份。
4. **路由规则保持极简**。默认「消息来自哪个 chat，回复就回哪个 chat」，不做花哨的跨平台转发。真有需求再显式加规则，别让路由层自作聪明。
5. **格式适配**。出站统一先转成一个中间 Markdown 子集，再由各插件降级到平台方言；超长内容 Discord 走分段或附件，Telegram 走分页。

## 踩坑点

- **回声循环**。Discord 插件最初没过滤 bot 自己发出的消息，Agent 的回复被再次当成输入路由回来，瞬间自言自语刷屏。所有入站消息必须按来源身份过滤，bot 自己的直接丢弃。
- **粒度对不齐**。Telegram 话题群和 Discord thread 不完全等价。我们让两者都映射成独立 session key——宁可多几个 session，也不要让不同话题串台。
- **限速差异**。Discord 对频道消息限速比 Telegram 严格得多。批量通知场景下 Telegram 侧毫无感觉的频率，Discord 侧开始静默丢消息。发送端要按平台配独立的速率预算和重试。
- **媒体不对称**。语音、贴纸、大文件两边支持差别很大。适配层对不支持的内容显式降级成「收到一条语音（未转写）」，比让 Agent 硬猜强。

## 可复用建议

- 路由和适配做成独立插件，别写进 Agent 主逻辑，加平台只动插件层；
- 入站消息带幂等键（platform + message id），重试去重靠它；
- 灰度顺序：先单向推送 → 再加命令 → 最后双向，每步观察一周；
- 日志始终带 `platform:chat_id:user` 标签，排障时一眼定位消息从哪来、回哪去。

## 总结

一个 Agent 服务两个平台，难点不在接入，而在会话隔离、身份一致、格式适配这三件枯燥的事。把它们压进一层薄插件，Agent 本体对平台保持无感知，之后再加 Slack 或 Matrix，也只是多写一个适配器而已。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/736a5ace7b511903.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/a955840a8b69b067.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/1ba8abf5caaa0412.png)

