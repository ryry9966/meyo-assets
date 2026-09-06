---
title: 一个 Agent 吃两条渠道：Telegram + Discord 跨平台消息路由实录
feedId: 36327
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我的 Agent 最初只挂在 Telegram 上，服务自己和几个朋友。后来协作迁到了 Discord，第一反应是再开一个 bot——但很快就发现不对：两份记忆、两套配置、双倍 token 消耗，而且同一个问题两边答案开始漂移。这篇记录我怎么用 OpenClaw 的 gateway 做成「一个大脑、两条渠道」。

## 问题：为什么不是开两个 bot 就完事

- **记忆分裂**：同一个人在两边聊，Agent 把他当成两个人。
- **格式差异**：Discord 的 markdown 语法在 Telegram 会原样露出；长度限制一边 2000 一边 4096。
- **触发逻辑不同**：Discord 群里必须 @ 才回，Telegram 私聊希望全收。
- **附件差异**：两边媒体的 URL 结构、时效、大小限制都不一样。

## 做法

**1. 单实例、多渠道适配器。** 一个 gateway 进程，Telegram 和 Discord 各配一份 token，指向同一个 agent 和同一个 workspace。不要起两个实例再做同步——那是分布式一致性问题，不是路由问题。

```yaml
channels:
  telegram:
    botToken: ${TELEGRAM_BOT_TOKEN}
  discord:
    botToken: ${DISCORD_BOT_TOKEN}
    intents: [message_content]
```

**2. 会话键显式设计。** 默认用 `agent:main:telegram:<chatId>` 和 `agent:main:discord:<channelId>` 天然隔离；需要打通时建一张 identity map，把两平台用户 ID 映射到统一 UID，再指向同一 session。建议默认隔离、按需合并，别一上来全共享——上下文串台比记忆分裂更难收拾。

**3. 渲染与归一化分离。** 入站把两边消息统一成内部 message 对象（文本、附件、发送者、session key）；出站每个渠道一个 renderer，Discord 出 markdown，Telegram 出 HTML。超长回复按段落结构切块，代码块不允许跨块。

**4. 触发与媒体。** Discord 配 mention-only，Telegram 私聊直通、群聊 mention。附件先落盘到 workspace 临时目录再给 Agent 路径，别把平台 URL 直接丢进上下文——Discord CDN 链接过期很快。

## 踩坑点

- Discord 开发者后台没开 **MESSAGE CONTENT intent**，bot 收到的全是空消息，白查了一小时。
- 测试时起过第二个实例，polling 冲突导致 Telegram 收不到消息。确认任何时刻只有一个实例持有 token。
- 限速不对称：Discord 有频道级限速，Telegram 有聊天级限速，连发回复必须排队，否则 429。
- 按字数硬切长回复会切碎代码块，对端渲染直接报错。按块结构切，别按长度切。

## 可复用建议

- **渠道层做薄**：只做归一化和渲染，业务逻辑一律不塞进去。
- 路由表（哪个群、哪个人、走哪个 agent）放配置文件，不写死在代码里。
- 日志里永远带 `platform + session key`，跨平台问题基本靠这两行定位。
- 写个冒烟脚本：同一条 prompt 打两条渠道，diff 两边的行为和格式。

## 总结

这件事本质不是「接两个平台」，而是在边界上把渠道差异消化掉，让 Agent 只面对统一的内部消息模型。归一化、会话键、渲染分离三件事做扎实之后，加第三、第四个渠道，基本就是复制粘贴一份适配器配置的工作量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/5f76760374384453.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/cd7698d33de4827a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a52881d64481cc87.png)

