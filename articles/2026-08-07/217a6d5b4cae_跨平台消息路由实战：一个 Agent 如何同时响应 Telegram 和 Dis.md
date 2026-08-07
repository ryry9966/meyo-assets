---
title: 跨平台消息路由实战：一个 Agent 如何同时响应 Telegram 和 Discord
feedId: 31994
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在 OpenClaw 社区里，很多朋友习惯把自动化流程挂在一个 Agent 上，让它监听消息、执行动作。当只需要支持一个平台时，事情很简单：对接一个 Bot API，写个轮询或 Webhook，拿到消息交给 LLM 或脚本处理就行。

问题出在“第二平台”加入之后。常见场景是：同一个助手既要回复 Telegram 私聊，又要在 Discord 频道里响应用户。早期做法是拆成两个服务，各跑各的，但这样业务逻辑会分裂，状态和上下文也难以共享。后来我们尝试用一个 Agent 进程同时接入两个平台，让它在统一的消息模型下工作。这篇文章就是那次实践的总结，重点放在工程层面——如何让两套通信机制和平共处，以及哪些地方容易踩坑。

---

## 问题拆解

跨平台消息路由的核心矛盾有三个：

1. **消息格式不同**  
   Telegram 的消息结构里有 `chat_id`、`message_thread_id`、`reply_to_message`；Discord 的则是 `channel_id`、`guild_id`、`message_reference`。而 Agent 内部不应该关心这些。

2. **连接方式不同**  
   Telegram 推荐长轮询（getUpdates）或 Webhook，Discord 强制使用 WebSocket 网关。一个进程里既要维护 WebSocket 心跳，又要收 HTTP 回调或者跑轮询循环，并发模型很容易搞乱。

3. **回复路径不同**  
   Telegram 回复需要原路返回到对应 chat，Discord 则需要知道是频道消息还是 DM。尤其当 Agent 内部触发了延迟任务（例如定时提醒），必须在任务上下文里保存正确的路由键。

清楚了这些问题之后，方案就比较好组织：**统一消息抽象 → 平台适配器 → 路由分发 → 回复适配**。

---

## 设计与实现

### 1. 统一消息模型

我们定义了一个与平台无关的内部消息对象：

```python
class UnifiedMessage:
    platform: str          # "telegram" 或 "discord"
    user_id: str
    channel_id: str
    text: str | None
    attachments: list[str]
    reply_to: str | None
    raw_meta: dict         # 保留原始引用
```

所有平台消息在进入 Agent 核心前，都会先被转换成这个结构。核心只依赖 `UnifiedMessage`，返回的回复也是一个类似的 `UnifiedReply`，包含文本和可选附件。

### 2. 平台适配器

每个平台封装为一个 Adapter，负责三件事：接收推送、转成统一格式、执行最终的 API 回复。

- **Telegram Adapter**：我们选了 Webhook 方式，因为进程内 WebSocket 已经占了 Discord，再加一个轮询循环会引入不必要的阻塞。Webhook 的回调直接抛到内部的 `asyncio.Queue`，由消费协程处理。
- **Discord Adapter**：使用 `discord.py` 的 `Client`，注册 `on_message` 事件。同样把消息丢进同一个队列。

两者共用一个消息队列，避免在事件回调里直接调用 Agent 核心（防止阻塞 WebSocket 或 Webhook 响应）。Agent 核心是一个独立的任务，从队列中取消息、处理、再把回复扔进另一个“出站队列”，由平台 adapter 各自消费并调用对应 API 发送。

### 3. 路由与回复

Agent 核心在处理完请求后，会生成 `UnifiedReply`，其中 `target` 字段沿用原消息的 `channel_id`。出站逻辑如下：

- 如果 `platform == "telegram"`：调用 `sendMessage` 并带上 `chat_id`，必要时处理 Markdown 转义。
- 如果 `platform == "discord"`：判断 channel 类型，调用 `channel.send()` 或 `user.send()`。

关键点在于，延迟回复（比如异步任务完成后发送结果）需要在创建时把 `platform` 和 `channel_id` 序列化到任务参数里，不能依赖线程局部变量。

---

## 踩坑点

### 1. Webhook 与 WebSocket 同进程的资源竞争

一开始我们把 Telegram Webhook 设成 Flask 子线程，Discord 用 `discord.py` 的 `run()` 阻塞主线程。结果 Webhook 收不到请求——因为 Flask 绑定的端口被内部防火墙或事件循环冲突影响。后来统一用 `aiohttp` 作为 HTTP 服务器，与 Discord 的 `Client` 共享同一个事件循环，问题解决。**建议**：跨平台适配器尽量用同一个 async 框架，避免多线程 + 多事件循环的噩梦。

### 2. Discord 的 Intent 与 Telegram 的隐私模式

Discord 需要显式启用 `message_content` Intent，并在开发者后台打开开关。忘记这一步，`on_message` 永远收不到文本内容，只会拿到空消息，而日志里看不出明显错误。Telegram 则需要把 Bot 设置为 **非隐私模式**，或让用户先给它发送 `/start`，否则 Bot 无法主动发起会话。两个平台的“沉默失败”很容易浪费半天调试时间。

### 3. 消息去重

当同一个用户同时在两个平台上跟 Agent 互动时，可能会出现“逻辑重复”的错觉——比如用户在 Telegram 发了 `/status`，又在 Discord 发了同样的命令。这其实是正常的，但需要 Agent 上下文隔离。我们用 `user_id + platform` 作为会话键，区分不同平台的同一用户。

### 4. 速率限制

Telegram 每秒最多 30 条消息，Discord 也有限流。出站队列加了个简单的 token bucket，避免短时间大量回复被平台拒绝。不能依赖 try-except 重试，因为重试风暴会加剧问题。

---

## 可复用建议

- **先定义消息契约，再接入平台**：哪怕开始时只支持一个平台，也用 `UnifiedMessage` 包裹原始消息。第二个平台接入时成本会低很多。
- **出站与入站解耦**：用队列把“接收消息”和“发送回复”完全分开，能极大减少并发问题，也方便加监控。
- **保留原始引用**：`raw_meta` 字段看起来很丑，但在需要特殊处理（例如 Discord 的 embed 或 Telegram 的 inline keyboard）时，它是救命的逃生口。
- **做好优雅关闭**：进程退出前，等待出站队列排空、主动关闭 WebSocket 和 HTTP server，避免消息丢失。

---

## 总结

让一个 Agent 同时服务 Telegram 和 Discord，本质上是在做“平台抽象层”的工程。难点不在于调用 API 本身，而在于统一消息模型、协调不同连接模式的并发、以及稳定地处理错误和限流。只要把这层抽象做好，后续再接入 Slack、Matrix 甚至更冷门的平台，都只是新增一个 Adapter 的事。

在 OpenClaw 的生态里，这类跨平台消息路由很适合封装成 Skill 或 MCP Server，让其他 Agent 直接复用。希望这份实战记录能帮你在多平台接入时少走些弯路。

---

