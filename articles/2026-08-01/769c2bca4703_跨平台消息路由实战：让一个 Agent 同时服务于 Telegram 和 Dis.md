---
title: 跨平台消息路由实战：让一个 Agent 同时服务于 Telegram 和 Discord
feedId: 31256
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

很多自动化实践者都会遇到同一个问题：我们的用户或社区分布在 Telegram 和 Discord，但核心 Agent 只有一套逻辑。重复部署既浪费资源，又难以保证行为一致。理想情况是 **一个 Agent 实例，同时响应来自两个平台的消息**，内部无感切换，对外表现一致。

在 OpenClaw 生态下，平台适配通常被抽象为 `Adapter` 或 `Transport`，但真正跑通跨平台消息路由时，会遇到不少工程细节，本文记录一次从零到稳定运行的实践。

## 挑战拆解

不同 IM 平台的消息模型差异巨大：

- **鉴权方式**：Telegram 依赖 Bot Token + Webhook / Long Poll，Discord 使用 Gateway Intents + REST API。
- **消息结构**：Telegram 消息体包含 `chat.id`、`message_thread_id`、键盘 inline keyboard；Discord 则有 `channel_id`、`guild_id`、交互组件、嵌入 (embed)。
- **限制条件**：Telegram 单条消息上限 4096 字符，Discord 是 2000（内容）或 6000（embed 描述），且各有不同的速率限制。
- **交互组件**：Telegram 的 inline button、ReplyKeyboard 和 Discord 的 Button、Select Menu 完全不同。

## 实现步骤

### 1. 消息归一化

在 Agent 入口定义统一的 `InternalMessage`：

```typescript
interface InternalMessage {
  platform: 'telegram' | 'discord';
  userId: string;       // 平台内唯一 ID
  chatId: string;       // 对话标识（可能跨频道/群组）
  threadId?: string;    // 用于 Telegram 话题组或 Discord 线程
  text: string;
  attachments?: Attachment[];
  raw: any;             // 保留原始消息，方便扩展
}
```

每个平台的 Adapter 负责将收到的 Webhook 事件或 Gateway 消息转换为 `InternalMessage`，然后交给 Agent 处理。

### 2. 双平台接入配置

在 OpenClaw 中，可以分别注册两个 Transport：

- **Telegram Transport**：通过 webhook 接收更新，收到消息后触发回调。
- **Discord Transport**：通过 Discord.js 或直接监听 Gateway 的 `messageCreate`，同样转为内部消息。

示例伪代码：

```typescript
agent.onMessage(async (msg: InternalMessage) => {
  const response = await agent.process(msg);
  // 根据 platform 调用对应发送函数
  if (msg.platform === 'telegram') {
    await telegramAdapter.send(msg.chatId, response.text, response.options);
  } else {
    await discordAdapter.send(msg.chatId, response.text, response.options);
  }
});
```

### 3. 响应发送适配

分别实现两个 Adapter 的 `send` 方法，处理消息分片和富文本转换：

- **长消息拆分**：根据各平台字符限制切成数组，依次发送。Telegram 可自动合并为连续气泡，Discord 可放在一个 embed 内（仍受长度限制）。
- **按钮/组件映射**：Agent 返回的交互组件应统一为抽象 `Action[]`，再由 Adapter 渲染为 Telegram InlineKeyboard 或 Discord ActionRow。
- **附件处理**：将内部附件类型映射为 Telegram 文件上传或 Discord 嵌入。

### 4. 去重与回环防护

当 Agent 同时在多个群组或频道中时，可能出现自己发的消息被再次接收。我们采取二层过滤：

- **发送方 ID 过滤**：在 Adapter 内忽略来自 bot 自身的消息（Telegram 可通过 `message.from.id` 判断，Discord 通过 `message.author.id` 判断）。
- **去重缓存**：对短时间内完全相同的 `(userId, text, timestamp)` 做幂等处理，防止 Webhook 重复投递（Telegram 偶发）。

## 踩坑记录

**坑1：Telegram 的速率限制极其隐蔽**  
当短时间发送大量消息时，Telegram 返回 HTTP 429，但不会立即告知 `retry-after`。我们曾因为批量通知导致部分消息丢失。最终实现了一个简单的令牌桶队列，默认 30 条/秒，并在 429 时延期重试。

**坑2：Discord Gateway Intent 配置遗漏**  
开始只有 GUILD_MESSAGES 和 MESSAGE_CONTENT，结果私信（DM）收不到。需要额外启用 DIRECT_MESSAGES Intent，才能让 Agent 响应私聊，这在文档中不够显眼。

**坑3：线程/话题的支持缺失**  
Telegram 的超级群话题下，`message_thread_id` 必须回传才能在正确子线程回复；Discord 的线程也需指定 `thread_id`。如果处理不当，回复会跑到主频道，造成上下文混乱。解决方案是在 `InternalMessage` 中加入 `threadId`，并在回复时原样携带。

## 可复用建议

- **抽象 Adapter 接口**，不要直接耦合具体平台 SDK。后期想增加 WhatsApp 或 Slack 时只需实现同样的 interface。
- **使用消息队列解耦接收与发送**。尤其是处理长时间 Agent 推理时，避免阻塞 webhook 响应（Telegram webhook 要求快速返回 200）。
- **集中管理配置**。将各平台的 Token、Webhook Secret、Channel 映射表放入统一配置文件，方便切换环境。
- **添加健康检查端点**，分别检查各 Adapter 状态。我们遇到过 Discord WS 断线而 Agent 仍在运行的情况，加一个简单的状态探针就能提前告警。

## 总结

一个 Agent 同时服务 Telegram 和 Discord，本质上是一道“异构系统统一路由”的习题。核心在于消息模型归一化、平台差异的隔离适配，以及对可靠性的额外设计。在 OpenClaw 的架构下，这部分完全可以被通用化，不必为每个 Agent 重复造轮子。当跨平台路由稳定后，你会发现用户数据、对话历史、插件能力都能真正复用，维护成本不升反降。

---

