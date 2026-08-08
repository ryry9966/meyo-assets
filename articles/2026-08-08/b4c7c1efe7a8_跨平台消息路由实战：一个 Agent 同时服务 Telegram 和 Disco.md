---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 32102
source: 综合讨论
publishedAt: 2026-08-08
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

在社区小团队或个人运维的 Agent 场景里，常常需要让同一个 Agent 实例同时响应来自 Telegram 和 Discord 的消息。比如：

- 同一个问答机器人，在 TG 群里查文档、做摘要，在 Discord 里也响应 slash command 或 @提及。
- 通过 OpenClaw 接入 LLM，希望两个平台的用户得到一致的体验，又能对不同平台做定制（如命令前缀、消息格式）。

然而现实中会碰到几个典型问题：两个平台的 API 模型完全不同，消息格式、身份标识、回话管理也都各自为政。如果直接在一个进程里混杂处理，代码很快就变得难以维护，甚至因为一个平台的限流或断连影响另一个渠道。

本文记录一种工程化的做法：利用消息总线（Bus）模式，将不同平台的消息统一为内部事件，然后交给同一个 Agent 处理，再把回复路由回对应平台。实现语言以 TypeScript 为例，但思路同样适用于 Python 等。

## 核心问题

1. **消息格式异构**  
   Telegram 有 `message.text`、`callback_query`，Discord 有 `interaction`、`message`，还有附件、按钮、嵌入等不同载体。
2. **身份与权限映射**  
   用户 ID、群组/频道 ID、权限需要在内部抽象成统一的 `Sender`、`Channel` 结构，否则 Agent 的权限判断和会话管理会变得耦合。
3. **回复方式的差异**  
   Telegram 可以用 `sendMessage` 甚至流式编辑（长轮询或 webhook），Discord 的 Interaction 则要求在 3 秒内回复，之后才能用 `followUp`。
4. **连接模型不同**  
   Telegram Bot 可以用长轮询或 webhook，Discord 一般用 Gateway WebSocket + HTTP Interaction。同时维持两种长连接并保持稳定性，对错误处理和重连策略有要求。

## 做法 / 步骤

### 1. 定义统一的内部事件

为所有入站消息定义一个最小接口：

```ts
interface InboundEvent {
  platform: 'telegram' | 'discord';
  eventType: 'message' | 'command' | 'interaction';
  sender: {
    id: string;
    name: string;
    platformSpecific: Record<string, unknown>; // 用于平台特有逻辑
  };
  channel: {
    id: string;
    type: 'dm' | 'group' | 'guild';
    platformSpecific: Record<string, unknown>;
  };
  content: {
    text: string;
    attachments?: { url: string; type: string }[];
  };
  raw: unknown; // 保留平台原始对象，以备不时之需
}
```

每个平台适配器负责将自己的原始消息转换成 `InboundEvent`。例如 Telegram 适配器需要：

- 判断 `message.chat.type` 来得到 `channel.type`
- 将 `message.from` 映射到 `sender`
- 处理 `entities` 中的命令实体，决定是否标记为 `command` 事件

Discord 适配器则需要：

- 区分 `interactionCreate` 和 `messageCreate`
- 从 `interaction` 中解析出 subcommand 和选项
- 注意 interaction token 的有效期，将 token 塞进 `raw` 以便后续回复

### 2. 构建统一的消息总线与 Agent 路由

用一个事件发射器（如 EventEmitter 或 RxJS Subject）作为内部总线。所有平台适配器连接总线，将 `InboundEvent` 推送上去。

Agent 侧订阅总线，拿到事件后做统一处理：

```ts
bus.on('inbound', async (event: InboundEvent) => {
  const response = await agent.handle({
    userId: event.sender.id,
    channelId: event.channel.id,
    platform: event.platform,
    content: event.content.text,
  });
  // response 里带回了目标平台和回复内容
  bus.emit('outbound', {
    target: event,        // 原入站事件，包含回复所需的上下文
    responseContent: response,
  });
});
```

关键点在于 `response` 不直接触发平台特定的发送函数，而是再通过总线分发出去，由各个平台适配器监听 `outbound` 事件，根据自己的 `event.platform` 判断是否由自己处理。

### 3. 平台适配器实现回复

- **Telegram 回复**：直接使用 `bot.telegram.sendMessage(chatId, text)`。如果需要流式输出，可以维护一个消息 ID，用 `editMessageText` 逐步更新。
- **Discord 回复**：  
  如果原事件是 `interaction`，必须用 `interaction.reply()` 或 `interaction.deferReply()` 在 3 秒内完成首次响应，之后用 `interaction.followUp()` 发送后续内容。若是普通消息（Message）事件，直接用 `message.reply()`。

因此，在 outbound 处理中需要根据 `event.raw` 判断回复方式：

```ts
if (event.platform === 'discord') {
  const interaction = event.raw as DiscordInteraction;
  if (!interaction.replied && !interaction.deferred) {
    await interaction.deferReply(); // 争取时间
  }
  await interaction.followUp({ content: text });
}
```

### 4. 连接与错误隔离

两个平台的长连接各自独立封装：

- **Telegram**：使用 `grammy` 或 `telegraf`，启动长轮询，在 `error` 事件中处理 `401`、网络超时，并实现指数退避重连。
- **Discord**：用 `discord.js` 维护 WebSocket 连接。处理 `shardError`、`error`，并监听 `shardDisconnect` 进行重连。

**关键：** 不要让其中一个平台的崩溃或卡死影响另一个。可以为每个适配器创建独立的 async context（如用 `Promise.allSettled` 启动），并确保总线内部有错误边界，不会因为一个事件的 handler 抛出异常而丢失其他事件。

## 踩坑点

1. **Discord Interaction 超时问题**  
   如果你没有在 3 秒内调用 `reply()` 或 `deferReply()`，Discord 会回报 “Unknown interaction” 错误。Agent 处理可能耗时较长（特别是 LLM 生成），所以**必须立即 defer**，然后通过 `followUp` 发送结果。对于一些 slash command，还可以用 `showModal` 或 `deferUpdate` 等方法，视情况而定。

2. **Telegram 长轮询的锁死**  
   如果 Agent 处理某个 TG 消息时抛出未捕获异常，可能导致整个 `grammy` 更新循环中断。需要在 adapters 内用 try/catch 包裹所有事件处理逻辑，并在 catch 中记录日志后继续轮询。

3. **身份映射不一致**  
   TG 用户 ID 是数字，Discord 用户 ID 是 Snowflake（字符串）。拼接统一 ID 时建议采用 `platform:uniqueId` 格式，避免碰撞。例如 `tg:123456`，`ds:9876543210`。

4. **消息附件和特殊实体**  
   一些平台特有功能（如 TG 的 inline keyboard、Discord 的 embed）如果需要在两个平台间互通，复杂度会急剧上升。建议先在 Agent 回复中限制为纯文本或 Markdown，后期再按需扩展。

## 可复用建议

- **所有适配器都实现相同的接口**：`initialize()`, `shutdown()`, 以及统一的 `sendMessage(channelId, text, options?)`。这样未来接入新平台（如 WhatsApp、Slack）时，只需按统一模式编写适配器。
- **采用插件式注册**：在 OpenClaw 或自建框架中，可以用配置文件或依赖注入来加载适配器。比如：
  ```ts
  const adapters = config.platforms.map(PlatformAdapterFactory.create);
  await Promise.allSettled(adapters.map(a => a.initialize()));
  ```
- **确保 Agent 无状态**：Agent 不应直接依赖平台特定对象，只接收 `userId`、`text` 等基本参数。平台逻辑完全在适配器和总线层隔离，这样 Agent 可以轻易替换底层模型而无需改动平台代码。
- **可观测性**：为每个入站和出站事件打印结构化日志，包含 `platform`、`eventId`、`latency` 等字段。可以快速定位是哪个平台出现了延迟或异常。

## 总结

通过统一消息总线 + 平台适配器的模式，可以在不侵入 Agent 核心逻辑的前提下，让一个 Agent 同时服务 Telegram 和 Discord。这样做带来的好处是：

- Agent 无需关心消息来源，专注处理对话和业务逻辑。
- 平台特性被封装在适配器中，异常隔离做到位。
- 后续增加新渠道的成本很低，只需写一个新的适配器。

对于已经使用 OpenClaw 做编排的团队，可以将这种模式作为一种标准的工程实践，让机器人真正在多平台间“活”起来，而不是各自为政的碎片化实现。

---

