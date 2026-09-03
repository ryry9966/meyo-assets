---
title: 一个 Agent 守两个平台：Telegram + Discord 消息路由实践
feedId: 35945
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

最开始只接了 Telegram，个人使用没有问题。后来协作迁到 Discord，希望在频道里 @ 一下就能得到回答。第一反应是再起一个实例，但很快暴露问题：两份配置要同步维护、技能和提示词逐渐漂移、模型费用翻倍，同一个人在两个平台的上下文也完全割裂。回到 OpenClaw 本身的结构看，网关天然支持多 channel 接入，真正要做干净的是路由、会话和格式化这三件事。

## 问题

1. **会话边界**：Discord 频道和 Telegram 群是不同对话，上下文必须隔离；完全隔离又导致同一个人两边各问一遍。
2. **输出差异**：Telegram 单条 4096 字符、对 MarkdownV2 转义极其敏感；Discord 上限 2000、markdown 方言不同。同一份回复直接转发必出事。
3. **接入机制不同**：Telegram 走 long polling，Discord 走 gateway WebSocket，断线重连和限流完全是两套逻辑。

## 做法

结构是「单网关 + 双 channel + 薄适配层」：

1. **单实例接入双渠道**：同一个 gateway 配置里声明 `channels.telegram` 与 `channels.discord`，共用一个 agent、一套技能和模型配置，只跑一个进程。
2. **会话键按平台隔离**：约定 `platform:scope:id`，如 `telegram:group:-100xx` 和 `discord:channel:123xx`，每个对话一份独立上下文。跨平台共享的信息放长期记忆，按用户身份关联，不塞 session。
3. **触发规则分开**：Telegram 私聊直答、群聊走白名单加命令前缀；Discord 只在被 @ 或指定频道响应，避免在服务器里抢话。
4. **格式化下沉适配层**：Agent 只产出标准化消息块（文本段、代码块、媒体引用），出站由 adapter 负责转义和按平台上限拆分；代码块不跨条硬切，宁可截断并提示。
5. **媒体单独处理**：Telegram 的 file_id 下载后经 Discord 重新上传，不试图复用链接。

## 踩坑点

- **双实例抢连接**：迁移期间新旧实例并跑，Discord 同一 token 两个会话互踢、反复掉线；Telegram `getUpdates` 也报 409。结论：一个渠道同一时刻只允许一个消费者。
- **ID 一律按字符串存**：Discord snowflake 和 Telegram chat id 都超出 32 位，当数字处理迟早踩精度坑。
- **限流**：Discord 单频道约 5 条/5 秒，Telegram 单群约 1 条/秒，拆分长消息要加发送间隔。
- **转义**：MarkdownV2 的转义矩阵不值得硬啃，统一改用 HTML parse mode，adapter 只做一次实体转义，错误率几乎归零。

## 可复用建议

- 平台差异全部压进 transport 层，Agent 层永远不知道对面是谁——这是方案能长期维护的前提。
- 渠道级独立启停：改 Discord 配置只 reload 该 channel，Telegram 不掉线。
- 两个渠道打独立日志标签，分别统计延迟和失败率，排障快一个量级。
- 白名单先收紧后放开，进公开服务器前先在私有频道跑一周。

## 总结

一个 Agent 同时服务两个平台，OpenClaw 在结构上直接支持，工程量不在接入，而在会话策略、格式拆分和连接稳定性。把差异隔离在适配层，其余保持一份代码一份配置，维护成本就可控。需要配置模板的可以跟帖，我贴一份脱敏后的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/c4bec3aa597b43c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/4fc3a2b0d8b422bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/d5e04630a513e8f7.png)

