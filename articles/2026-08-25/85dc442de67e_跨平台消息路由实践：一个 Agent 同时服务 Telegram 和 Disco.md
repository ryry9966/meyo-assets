---
title: 跨平台消息路由实践：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34660
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

很多 OpenClaw 用户在给 Agent 接 IM 时，会先做 Telegram bot，等 Discord 有需求后再复制一份逻辑，起第二个进程。这样看起来两个平台都在跑，但问题很快出现：同一套工具和提示词要改两处，状态不同步，TG 私聊里能用的功能在 DC 频道里偶尔失效，排障时还要查两套日志。

更合理的思路是：Agent 核心只处理统一消息事件，平台差异交给接入层消化。一个 Agent 进程同时服务 Telegram 和 Discord，本质不是“两个 bot 调同一个模型”，而是做一层克制的事件归一和发送适配。

## 问题

直接让两个平台的消息打到同一个 webhook 是行不通的。Telegram 和 Discord 在消息结构、Markdown 语法、长度限制、附件上传、编辑删除、限流和身份体系上都不同。比如 TG 消息长度上限约 4096 字符，Discord 单条 2000；TG 的 MarkdownV2 转义规则很严格，Discord 相对宽松但代码块和链接写法不完全一样；TG 有 flood control，Discord 有全局 rate limit。如果这些差异散落在 Agent 逻辑里，代码会很快变成 `if platform == "telegram"` 的堆叠。

## 做法

### 1. 定义统一事件模型

所有平台消息先被规范成内部结构。入站消息可以简单定义为：

```json
{
  "platform": "telegram",
  "channel_id": "-100123",
  "user_id": "123",
  "text": "查一下构建状态",
  "attachments": [],
  "reply_to": null,
  "ts": 1710000000
}
```

出站消息类似：

```json
{
  "platform": "discord",
  "channel_id": "456",
  "text": "构建状态: passing",
  "attachments": [],
  "reply_to": "msg_789"
}
```

Agent 只消费和产生这两种内部结构，不关心具体平台。

### 2. 平台接入层

Telegram 侧建议先用 `getUpdates` 轮询，Discord 侧用 Gateway 连接。不要一上来就做 webhook 和 Interactions 签名校验，除非你已经对这两套体系很熟。先用稳定库跑通消息收发，例如 TG 用 aiogram，Discord 用 discord.py 或 nextcord。

### 3. 队列与路由

所有入站消息进入一个内部队列，Agent worker 逐个消费。出站消息进入另一个发送队列，由各平台的 sender 负责推送。队列要设上限，避免某个平台刷屏拖垮另一个平台的响应。

### 4. 平台发送适配

发送前调用 `format_for_platform(outbound, platform)`，做三件事：长度分片、Markdown 转换、附件上传。TG 和 Discord 的附件流程完全不同，图片、文件要分别走各自的 API。

### 5. 会话键与提示词

会话键不建议只用 `user_id`，因为两个平台的用户 ID 不互通。群聊用 `platform + channel_id`，私聊用 `platform + user_id`。系统提示里可以注入一个 `platform` 字段，但不要维护两套完整 prompt，差异只应影响少量平台语义。

## 踩坑点

- **Markdown 差异**：不要直接透传用户输入。内部输出优先 plain text 或受控子集，再由适配器转换。TG 的 MarkdownV2 对 `_`、`*`、`[` 等字符敏感，Discord 相对宽容。
- **长度分片**：Discord 单条 2000 字符，TG 4096。分片函数要按段落或代码块边界切，避免把代码块切坏。
- **编辑与删除**：TG 支持编辑消息，Discord 的编辑/删除语义不同。如果还没想清楚产品行为，先不支持编辑，只记录原始消息 ID。
- **限流与重试**：Discord 的全局 rate limit 比 TG 更复杂。发送层要做指数退避并加 jitter，不要无限重试。
- **身份与权限**：管理员白名单必须按平台分别配置。不能假设 TG 里的某个 user_id 等于 Discord 里的某个 user_id。

## 可复用建议

把平台适配层做成 OpenClaw 的 transport 插件，不要写进 Agent 核心里。内部消息统一带 `platform` 字段，方便日志过滤和按平台返回。发送队列与 Agent 回复解耦，Agent 只负责生成 outbound，发送失败由发送层重试。上线前先在本地跑双平台 echo 测试，再逐步放开工具调用。最后，给每个平台记录消息量、失败重试数和发送延迟，跨平台问题基本都能从这些指标里看出端倪。

## 总结

跨平台消息路由的核心不是“一个模型服务两个 bot”，而是把平台差异关进适配层。Agent 只面对统一消息事件，Telegram 和 Discord 的 Markdown、长度、附件、限流等问题在平台层解决。这样做以后，增加第三个平台时，不需要改 Agent 核心，只需新增一个 transport。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/530af6cc6a5faffb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1fc73c0516740a74.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/477242da1d526958.png)

