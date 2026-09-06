---
title: 一个 Agent 同时接 Telegram 和 Discord：跨平台消息路由的落地记录
feedId: 36300
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

社区分散在 Telegram 和 Discord 是常态。最开始我在两边各跑了一个 Agent 实例，很快就出现典型症状：两边人格漂移、记忆不同步、同一套提示词要维护两份。目标于是很朴素：**一个 Agent 大脑，两个入口，消息能正确进来、正确回去**。

## 问题

真正的工作量不在"接上 SDK"，而在中间那层路由：

- 两边身份体系不同：Telegram 的 `chat_id/user_id` 和 Discord 的 `guild/channel/user` 对不上；
- 会话边界怎么切：按平台、按用户，还是按聊天群；
- 出站格式差异：Telegram MarkdownV2 转义 vs Discord markdown，4096 vs 2000 字符上限；
- 重试与去重：两边的投递机制都可能重复推送，不处理就会复读。

## 做法

核心思路是把"接入"和"大脑"拆开：渠道适配器只负责收发，统一成信封格式交给路由层。

**1. 统一消息信封。** 适配器把原始事件归一化成 `platform / chat_id / user_id / thread_id / reply_to / content / media`。Agent 只看信封，不关心来源。

**2. 路由表。** 用一份 YAML 声明哪些聊天映射到哪个会话空间：

```yaml
routes:
  - platform: telegram
    chat: "-1001234..."
    session: community-cn
  - platform: discord
    channel: "1122..."
    session: community-cn
    threads: inherit
```

**3. 会话策略。** 我选了"共享 workspace + 每聊天独立会话线程"：长期记忆和人格共享，短期上下文按聊天隔离。同一个人在两个平台出现，不会互相污染上下文，但知识是一致的。

**4. 出站渲染。** 回复先产出中性的结构化内容（段落 / 代码块 / 列表），再由各平台 renderer 落地：Telegram 走 MarkdownV2 并统一转义；Discord 按块切片，保持代码块完整。

**5. 幂等与限速。** 以 `platform:message_id` 作去重键，窗口 10 分钟；出站统一进队列，按平台令牌桶限速，超长回复分片发送。

## 踩坑点

- **MarkdownV2 转义是重灾区。** 普通文本里的 `_` `*` `~` 会直接 400，别手写转义规则，用现成库。
- **Discord 硬切字符会拦腰斩断代码块。** 按块切，单块仍超长时降级为附件发送。
- **重复投递。** 网络重试加重连逻辑，出现过同一条消息触发两次回复。去重键必须落在入站消息 ID 上，用时间戳必翻车。
- **媒体引用不通用。** Telegram 的 `file_id` 只对同一个 bot 有效，Discord 附件 URL 带签名会过期。媒体先落本地缓存，再交给 Agent。
- **回复语义不对齐。** Discord 的 reply 引用和 thread 与 Telegram 的 `reply_to` 语义不同，thread 内消息要显式映射，否则 Agent 会把追问当成新话题。

## 可复用建议

- 信封 schema 定下来就别再动，新增平台只写适配器；
- 渲染器和适配器彻底分离，格式问题归渲染器管；
- 上线前做录制回放：把真实入站事件存成 JSONL，回放验证路由和去重逻辑；
- 两边各留一个测试群，投喂格式刁钻的消息（嵌套代码块、下划线、emoji 混排）做冒烟测试。

## 总结

跨平台路由拆开看就是三件事：统一的入站信封、可配置的路由与会话映射、按平台的出站渲染与限速。这三层切干净之后，Agent 大脑完全不需要感知"自己在哪个平台说话"。这套结构稳定跑了一个多月，期间新增一个渠道只改了配置加约一百行适配器代码。会话切分策略大家各有取舍，欢迎在评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/bb5231170679e23e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a64c69f1b4d6b04f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a061d61c330a8aa6.png)

