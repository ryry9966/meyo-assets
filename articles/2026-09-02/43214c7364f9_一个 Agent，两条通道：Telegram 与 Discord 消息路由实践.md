---
title: 一个 Agent，两条通道：Telegram 与 Discord 消息路由实践
feedId: 35854
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

社区用户一半在 Telegram，一半在 Discord。最初图省事，起两个 bot 项目各自维护，结果两周后就露馅了：两边人问同一个问题得到不同答案，prompt 和记忆各一套，改一个 bug 要同步两处。后来做了一次重构：把 Agent 核心抽出来，平台侧只留薄薄的适配层，这篇文章记录整个过程。

## 问题

抽象前直接面对三类冲突：

1. **会话与记忆不统一**：同一个人两个平台两个身份，上下文完全割裂。
2. **消息格式不兼容**：Discord 的 Markdown 方言、Telegram 的 MarkdownV2/HTML 转义各有各的坑，长度上限一个 2000 一个 4096。
3. **事件模型不同**：Discord 走 Gateway 长连接、事件很杂；Telegram 走 long polling，限流策略也不同。

## 做法

**第一步，固化核心入口。** Agent 核心只暴露一个函数，事件结构统一：

```json
{
  "channel": "telegram | discord",
  "session_key": "tg:chat:12345",
  "user_ref": "tg:user:67890",
  "segments": [{ "type": "text", "body": "..." }],
  "reply_to": null
}
```

MCP 工具、记忆、插件配置全部挂在核心上，适配器完全不感知。

**第二步，锁死适配层职责**：入站归一化，出站渲染。入站把平台消息翻译成上面的事件；出站按 channel 选 Markdown 方言、拆分段落、上传附件。

**第三步，会话路由。** 群聊用 `channel:chat_id` 做 session_key，私聊用 `channel:user_id`。跨平台身份绑定做成可选命令（私聊发一次性码核验后共享记忆），默认两平台隔离，边界更干净。

**第四步，部署形态。** 单个 asyncio 进程跑两个常驻任务，中间一个带每通道令牌桶的出站队列做分发和限流。量大了再拆进程、上 Redis。

## 踩坑点

- **转义**：Telegram MarkdownV2 要转义十几个特殊字符，别手写正则，用库或直接 HTML 模式。
- **长回复切分**：按段落和代码块边界切，不要按字符数硬切，否则代码块被腰斩。
- **限流**：Discord 按通道限，Telegram 全局限且群内更严。出站必须有队列缓冲，否则一次 @全员 场景直接 429。
- **重复消费**：断线重连后两边都可能重放消息，事件里加幂等键（message_id 去重）。
- **消息编辑**：Discord 有 update 事件，不提前决定是否重新触发 Agent，同一条消息会被回答三次。

## 可复用建议

- 核心代码禁止 import 任何平台 SDK，出现一次就是污染，迟早返工。
- 事件 schema 做版本号，以后接 Slack、QQ 只是多写一个适配器。
- 本地开发挂一个"终端适配器"，事件直接打印到 stdout，调试不用在两个平台间横跳。
- 全链路加 correlation id，从入站事件到出站回复一查到底。

## 总结

跨平台消息路由本身不难，难的是克制——平台逻辑只要漏进核心一处，抽象就开始腐烂。适配层做薄之后，接新平台约等于一天的工作量，而 Agent 侧的 prompt、记忆、插件变更则全平台同时生效。这套结构的关键收益不在"支持两个平台"，而在于之后每加一个平台，边际成本都在递减。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/862efc4ec501d872.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/08a0b870ee5565ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f17cf5dc1f9fe553.png)

