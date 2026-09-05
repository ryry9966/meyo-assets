---
title: 一个 Agent 同时接 Telegram 和 Discord：跨平台消息路由的实现与踩坑
feedId: 36245
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我们的用户社群一半在 Telegram、一半在 Discord。早期做法是两个平台各跑一个 bot 实例，背后连同一个 Agent，但实际用起来很别扭：记忆不通、插件配置双份、每次升级要改两处、排障要在两套日志里来回切。

目标是收敛成单实例：一个 Agent 进程，同时服务两个平台，记忆与工具链完全复用。

## 问题

这不是"多填一个 bot token"的事，核心差异有四块：

1. **接入模型不同**：Telegram 走 long polling 或 webhook（HTTP 拉取），Discord 走 gateway WebSocket 长连接加心跳。
2. **身份不互通**：平台 user id 各自独立，同一个真人在两边是两个身份——记忆要不要打通、怎么打通需要明确策略。
3. **消息能力不对齐**：Markdown 方言不同、长度上限不同（Discord 2000 / Telegram 4096）、回复引用与文件上传机制也不同。
4. **出站限流规则不同**：高峰期裸发两边都会吃 429。

## 做法

架构上一句话概括：**平台差异全部收敛在 adapter 层，Agent 只面对统一的内部信封（envelope）**。

1. **定义统一信封**：`platform / channel_id / user_id / msg_id / reply_to / text / attachments / ts`，入站一律归一化成这个结构。
2. **每个平台一个 adapter**，负责入站归一化和出站渲染。Telegram 用 long polling，Discord 用 gateway WebSocket，都不需要公网回调地址，家用宽带 + 反代就能跑。
3. **会话键设计**：默认 `session_key = (platform, channel_id, user_id)`；再可选一层 identity map 把两边 user 绑到同一内部用户，共享长期记忆，但短期上下文仍按平台隔离，避免跨平台串台。
4. **出站统一走队列**：按 `(platform, channel)` 维度做令牌桶；超长文本先按各平台上限做逻辑切分，再按方言渲染（Telegram 用 HTML parse mode，Discord 用它支持的 markdown 子集）。
5. **幂等与去重**：出站消息带客户端生成的 nonce；重连恢复后以最近 `msg_id` 做水位线，防止补拉阶段重复投递。

简化配置大致长这样：

```yaml
channels:
  telegram:
    adapter: telegram
    mode: polling
  discord:
    adapter: discord
    mode: gateway
routing:
  shared_memory: true        # 身份打通后共享长期记忆
  session_scope: per_channel_user
```

## 踩坑点

- **Markdown 方言**：Telegram 的 MarkdownV2 转义极其苛刻（`_ * [ ]` 等都要转义），Discord 又不支持部分扩展语法。结论：不要透传 Agent 原始输出，adapter 里各写一个 renderer，并对常见符号加单测。
- **切分时机**：先渲染再切分会把代码块切烂。正确顺序是先按平台上限在空行/代码块边界切逻辑块，再渲染。
- **Discord 重连风暴**：网络抖动后 gateway 会 resume，session 失效时触发全量重放。没做水位线的话，积压消息会被再喂一遍 Agent。我们用 `msg_id` 单调性判断直接丢弃。
- **Telegram 429**：尊重返回的 `retry_after`，不要自己写指数退避硬刚，越刚限流越久。
- **跨平台回声**：如果群里还跑着桥接（bridge）工具，Agent 的消息可能被桥回灌形成环。按 bot 自身 id 过滤即可。

## 可复用建议

- adapter 接口只暴露 `on_message / send / render` 三个方法，后续接 Slack、飞书的边际成本大约一天。
- 信封 schema 提前留好 `attachments` 和 `thread_id` 字段，将来支持论坛类频道不用改协议。
- 每通道独立 metrics（入站速率、出站失败数、429 次数），排障时一眼定位是哪边的锅。
- 灰度上线：先让 Agent 在新平台只读一周（记录不回复），验证过滤规则和会话键无误后再放开发言。

## 总结

跨平台路由的本质不是多接两个 token，而是三件事：把传输差异压进 adapter、把身份与会话策略想清楚、把出站当成一个带限流和幂等的投递问题来对待。做完之后，Agent 的记忆、插件和工具链对所有平台完全复用，维护成本从"两份"回到"一份"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/c633b0f2c9b75f70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9263648152cfdf5c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/41a5fc82d9c8daa7.png)

