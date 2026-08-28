---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 35005
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

“一个 Agent 同时服务 Telegram 和 Discord”并不是把两个 bot token 塞进一个进程就能稳定跑。尤其在 OpenClaw 这类可挂 MCP、插件、外部工具的 Agent 环境里，很多人先跑通一个平台，再复制一份配置接第二个平台，结果往往是上下文割裂、权限重复、消息串台。

更合理的目标是：让同一个 Agent 的决策、工具调用、记忆保持单实例，Telegram 和 Discord 只作为入口和出口存在。

## 问题

跨平台消息路由主要会碰到四类问题：

- **API 语义不同**：Telegram 是 chat_id + webhook/长轮询；Discord 是 gateway + channel_id/thread_id + interaction。
- **用户身份不同**：同一个自然人在两个平台有完全不同的 ID，权限和偏好需要显式映射。
- **回复必须回到来源会话**：否则两个平台的用户都会看到错乱回复。
- **超时与限流差异**：Discord slash command 要求在 3 秒内响应，Telegram webhook 也要快速返回，不能在里面跑长任务。

## 做法与步骤

### 1. 先定义统一消息模型

不要直接把平台 API 散落在 Agent 核心里。先定义统一的 InboundMessage / OutboundMessage：

```text
platform: "telegram" | "discord"
chatId: string
threadId?: string
messageId: string
senderId: string
text: string
replyToMessageId?: string
raw: object
```

输出消息也带 `routeKey`，例如：

```text
routeKey = platform + ":" + chatId + ":" + (threadId || "main")
```

Agent 核心只处理 InboundMessage，返回 OutboundMessage，由路由层负责投递。

### 2. 单 Agent 实例 + 双连接器

建议目录结构：

```text
connectors/
  telegram.ts
  discord.ts
  index.ts
core/
  agent.ts
  router.ts
```

`telegram.ts` 和 `discord.ts` 只做协议转换。`index.ts` 把两个连接器的标准化事件汇入同一个队列。Agent 从队列消费，处理完后通过 `router.ts` 调用对应连接器的 send / edit。

### 3. Discord interaction 单独处理

Discord 的 slash command、按钮点击不是普通 message，必须走 interaction 响应。收到 interaction 后应先 `deferReply` 或 `deferUpdate`，再把它转成 InboundMessage。把 interaction token 放进 `raw`，供后续 edit 使用。不要等 Agent 处理完才响应，否则 3 秒超时。

### 4. 显式身份映射

不要只拿平台 ID 做权限。维护一个映射：

```yaml
identity_map:
  - global_user: alice
    telegram_id: "111"
    discord_id: "222"
  - global_user: bob
    telegram_id: "333"
    discord_id: "444"
```

敏感命令只看 `global_user`，不要进入平台连接器判断。这样同一人在两个平台权限一致，也不会因为 ID 碰撞越权。

### 5. 命令与渲染统一

Telegram 用 BotFather 设置 command list，Discord 需要注册 slash command。核心把命令统一成 `{ name, args, source }`。渲染消息时不要直接拼平台 markdown，使用统一中间格式，例如仅支持加粗、代码块、链接，由各连接器转成平台格式。Discord 消息长度注意 2000，Telegram 4096。

### 6. 异步任务与重试

长任务放入本地队列：先回执，例如 Telegram 发 typing 或“处理中”，Discord 用 deferred response；完成后 edit 原消息或发新消息。对限流做指数退避，不要无脑重试。

## 踩坑点

- **把 Discord interaction 当 message**：按钮、斜杠命令会超时失效。
- **只用 chat_id 当唯一会话键**：Discord 的 thread 与频道需要合并成 route key，否则线程消息会回到频道。
- **直接透传平台 ID**：跨平台身份伪装或配置混乱。
- **在 webhook 里做长任务**：Telegram 会重试，Discord 会超时，导致重复触发同一命令。
- **忽略消息格式差异**：Telegram 的 MarkdownV2 转义很严格，Discord 的 markdown 又不同，统一渲染层能省很多奇怪 bug。
- **不处理幂等**：两个平台可能同时触发同类任务，需要按参数做短时去重。

## 可复用建议

- 连接器只做协议转换，不写业务策略。
- 身份映射放进配置文件或数据库，不要写死在代码里。
- 给每个连接器限制队列深度和单任务超时。
- 输出消息统一用 route key，不在 Agent 核心裸露平台特定字段。
- 新增平台时只新增 connector，核心与 router 不修改。

## 总结

跨平台消息路由的关键不是多接一个 token，而是把“入口”和“回复路由”从 Agent 核心中剥离。统一消息模型、显式身份映射、异步执行、平台渲染层，这四件事做完后，一个 Agent 同时服务 Telegram 和 Discord 才会稳定、可维护，而不是两个 bot 共用一个名字。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/52eca3d2dbec4f4a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/bcc94a11ab6943bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/3eb4d39c600e9a4c.png)

