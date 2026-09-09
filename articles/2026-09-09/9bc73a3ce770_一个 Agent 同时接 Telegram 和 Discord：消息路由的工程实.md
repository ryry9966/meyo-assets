---
title: 一个 Agent 同时接 Telegram 和 Discord：消息路由的工程实践
feedId: 36796
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

我们社区的用户分散在 Telegram 和 Discord 两边。早期偷懒，两个平台各跑一个 bot 实例，提示词和工具配置各自维护。三个月后两边行为明显分叉：一边接了知识库工具，一边没有，回答风格也对不上。这次改造的目标很朴素：**一个 Agent 内核，两个入口，行为一致。**

## 问题

这不只是“多接一个 API”的事。真正要解决的是四件事：入站消息如何归一化、会话上下文如何隔离、回复如何路由回正确的平台、以及两个平台在格式和限速上的差异如何兜住。

## 做法

1. **先定消息信封。** 内部统一 schema：`platform / chat_id / user_id / msg_id / text / attachments / reply_to / thread_id / is_mention`。适配器只做一件事：把平台原始 payload 翻译成信封，出站时反向翻译。
2. **适配层保持薄。** Telegram 走 long polling（内网部署不需要公网 HTTPS），Discord 走 gateway websocket，记得开启 Message Content Intent。适配器里不写任何业务判断。
3. **会话键设计。** `session_key = hash(platform, chat_id, thread_id)`。群聊与私聊天然隔离；Discord 的 thread 和 Telegram 的 forum topic 映射到第三段，避免跨串污染上下文。
4. **出站走队列。** Agent 回复携带来源信封，由 dispatcher 写入各平台独立的出站队列。队列负责分片、格式转换、限速重试，内核完全无感。
5. **渲染层按平台分流。** Telegram 侧用 HTML 模式，Discord 侧用原生 Markdown；代码块内容 passthrough 不转义。

## 踩坑点

- **长度限制不同。** Telegram 单条 4096 字符，Discord 2000。按段落分片，且要追踪代码围栏状态——在代码块中间截断是最难看的故障。
- **重复投递。** Discord 断线重连会重放事件，Telegram polling 重试也可能拿到重复消息。用 `(platform, msg_id)` 做幂等去重，一个带过期的 LRU 就够。
- **回复语境丢失。** 没映射 reply_to / thread 时，bot 会把串内回复发到主频道，对话直接断掉。这两段必须进信封，不能事后补。
- **延迟感知。** Agent 推理动辄数秒，不先发 typing 指示会像挂了。入站后立刻发 chat action，长任务中途再补一次。
- **触发策略。** 群里默认只响应 @mention，私聊全收，两个平台一致。否则刷屏速度取决于管理员的心情。

## 可复用建议

- 信封 schema 先行，适配器永远不带业务逻辑——这是将来加第三个平台（Slack、QQ）时唯一要守住的约定
- 出站统一过队列 + 每平台独立限速器，别在业务代码里 `sleep`
- 日志强制带 platform 标签，排障时按来源过滤能省一半时间
- 平台能力差异做成配置开关，比如贴纸只在 Telegram 侧启用
- 没有强需求别做跨平台身份合并，两边的 user_id 体系没有任何对应关系

## 总结

这轮改造的核心结论：Agent 本身是最简单的部分，八成工作量花在信封定义、出站格式化和限速兜底上。信封稳定之后，“多平台”从一个产品问题退化成一个适配器问题——这正是我们想要的状态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/83ad11dd28a53be6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a673ef1571ca2b7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a37af485aa6932c8.png)

