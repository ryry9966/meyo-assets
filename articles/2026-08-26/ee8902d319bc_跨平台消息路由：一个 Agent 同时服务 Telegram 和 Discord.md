---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34759
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

只接一个 IM 平台时，大部分问题都不明显：消息结构固定、回执方式单一、会话键直接用 `chat_id` 也够用。但一旦同一个 Agent 要同时服务 Telegram 和 Discord，两端协议、事件字段、消息长度、限流策略都不同。如果分别维护两套处理逻辑，很快会变成补丁堆叠。

跨平台消息路由的核心不是“接两个 API”，而是把平台差异压缩到 adapter 层，让 Agent 只面对一种内部消息格式。

## 问题

常见的问题不只是“能不能收到消息”，而是：

- Telegram 的 `update` 与 Discord 的 `message` / `interaction` 结构完全不同。
- Telegram 有 `chat_id`，Discord 有 `channel_id`、`guild_id`、`thread_id`，scope 概念不一致。
- 消息长度限制不同：Telegram 常规文本接近 4096，Discord 内容上限 2000。
- 格式不同：Telegram 的 `MarkdownV2` 与 Discord 的 Markdown 不完全兼容。
- 触发方式不同：Discord 的 slash command 需要响应 interaction，且通常要快速返回，否则会超时。

这些问题不适合堆在 Agent 主流程里处理，而应该在入口处统一抽象。

## 做法 / 步骤

### 1. Adapter 只做协议转换

给每个平台写一个 adapter，只负责：

- 接收平台回调；
- 转成统一 `InternalMessage`；
- 把处理结果按平台格式发回。

统一结构可以保持最小：

```ts
type InternalMessage = {
  platform: 'telegram' | 'discord';
  scopeId: string;      // Telegram: chat_id; Discord: channel_id
  userId: string;
  messageId: string;
  text: string;
  replyTo?: string;
  raw: unknown;
};
```

业务层只依赖 `InternalMessage`，不直接接触 `update` 或 Discord SDK 对象。

### 2. 会话键必须带平台前缀

不要直接用 `chat_id` 或 `channel_id` 作为 session key，否则两个平台可能撞出相同的 ID。建议：

```ts
const sessionKey = `${msg.platform}:${msg.scopeId}`;
```

如果希望同一用户跨平台共享上下文，先不要基于 `userId` 自动合并。跨平台身份绑定最好通过显式关联，例如用户在两个平台分别执行同一个绑定指令，而不是靠猜测 `userId` 一致。

### 3. 统一处理链

OpenClaw 里可以把 Agent 接到一个路由函数中：

```ts
const internal = adapter.normalize(platformPayload);
const session = sessions.get(`${internal.platform}:${internal.scopeId}`);
const reply = await agent.handle(internal, session);
await adapter.send(internal, reply);
```

这里不需要复杂框架，关键是保持 `normalize -> session -> handle -> send` 的单向流。插件、命令、权限检查都应在这个链路上做，而不是散落在两个平台入口处。

### 4. 发送逻辑按平台拆分

不要把同一个格式化函数复用到两个平台。例如：

- Telegram 使用 `parse_mode: 'MarkdownV2'`，需要对特殊字符转义。
- Discord 超过 2000 字符需要分片，或转成 embed；如果使用 interaction，要先 `deferReply` 避免 3 秒超时。

可以给 adapter 定义 `sendText()`, `sendChunked()`, `sendError()` 等基础方法，具体实现留在 adapter 内。

### 5. 配置与启动

同一个 Agent 进程内同时启动两个 adapter：

```bash
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
AGENT_ID=main
```

Telegram 本地调试可以用 polling，Discord 建议用 gateway 或 webhook。不要把 token 写进仓库，调试时保留 `verbose` 日志，但注意对 token 和用户内容脱敏。

## 踩坑点

- **Discord interaction 响应超时**：如果 Agent 处理超过 3 秒，需要先 `deferReply`，否则请求会失败。
- **重复投递**：webhook 可能重试，polling 可能收到同一消息多次。必须按 `platform + messageId` 做幂等去重。
- **Bot 收到 Bot 消息**：Telegram 里过滤 `from.is_bot`，Discord 里过滤 `author.bot`，否则容易形成回环。
- **Markdown 差异**：Telegram 的 `MarkdownV2` 对 `.`、`-`、`!` 等字符敏感；Discord 没有这么严格。不要共享同一套转义规则。
- **限流**：Discord 全局 rate limit 比较严格，Telegram 群组也可能限制转发频率。发送失败需做指数退避，不要无限重试。

## 可复用建议

- Adapter 接口保持小：`start()`、`stop()`、`normalize()`、`send()` 足够。
- 内部事件统一用 `platform + scopeId + messageId`，减少业务代码里的分支判断。
- 每个平台单独做健康检查：Telegram 用 `getMe`，Discord 用 gateway 状态或 ping。
- 日志保留原始 payload，但必须裁剪 token 和敏感字段。
- 先完成单平台稳定运行，打开全量日志，再接入第二平台；不要同时调试两个新平台。

## 总结

一个 Agent 同时服务 Telegram 和 Discord，本质上是一道消息归一化问题。Adapter 负责协议转换，内部消息负责统一模型，session key 负责上下文隔离，发送层负责平台差异。做好幂等、限流和格式转换之后，多平台接入不会增加 Agent 主流程复杂度。

不要过早抽象，也不要为了“全平台兼容”引入不必要的中间层。两个平台先跑通，再考虑扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/51f3a92341b3caf0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/a1884e289eb8564d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/cdf629a608e021f6.png)

