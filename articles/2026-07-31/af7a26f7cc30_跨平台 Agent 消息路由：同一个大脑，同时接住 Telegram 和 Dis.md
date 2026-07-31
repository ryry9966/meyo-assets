---
title: 跨平台 Agent 消息路由：同一个大脑，同时接住 Telegram 和 Discord
feedId: 31075
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景

在 OpenClaw 生态里，构建一个能处理多轮对话、调用工具链的 Agent 并不难。真正的麻烦出在“渠道”上：团队里有人只用 Telegram，另一拨人坚守 Discord，外部合作方可能通过第三方 Bot 接入。如果为每个平台维护一个独立的 Agent 实例，不仅浪费计算资源，还会产生状态不一致、配置难同步、提示词版本管理混乱等问题。

更合理的做法是：让**一个** Agent 通过统一的消息路由层，同时服务 Telegram 和 Discord。这篇文章记录了我在 OpenClaw 上实现这种跨平台消息路由的过程、遇到的坑，以及一些可复用的设计思路。

## 问题定义

将同一个 Agent 暴露给多个聊天平台，核心挑战不在于 SDK 的调用，而在于抽象出一层“平台无关”的消息管道。具体表现为：

1. **消息格式差异**：Telegram 的 `Message` 对象与 Discord 的 `Interaction` / `Message` 结构完全不同，包括附件、回复引用、按钮交互等元信息的表达方式都不一致。
2. **会话标识差异**：TG 使用 `chat_id`（含用户/群/频道），Discord 则区分 `guild_id`、`channel_id`、`user_id`，直接透传会导致会话管理混乱。
3. **响应方式差异**：TG 可以直接 `sendMessage`，Discord 的斜杠命令需要按 `Interaction` 回执，且对延迟有硬性限制（3 秒内必须回复初始 ACK）。
4. **平台限制**：消息长度、富文本格式、按钮数量、速率限制（尤其是 Discord 的全局 rate limit）各不相同，Agent 的原始输出需要根据目标平台做适配。

如果不在架构上隔离这些差异，Agent 的核心逻辑很快就会被平台特性污染。

## 实现步骤

以下基于 OpenClaw 的插件体系，给出可落地的实现路径。

### 1. 定义统一消息结构

首先建立一个平台无关的标准化消息对象（以 TypeScript 为例）：

```ts
interface UnifiedMessage {
  sessionId: string;      // 全局唯一会话 ID，由适配器生成
  platform: 'telegram' | 'discord';
  senderId: string;       // 平台侧用户标识
  content: string;        // 纯文本内容
  attachments: { type: string; url: string }[];
  raw: unknown;           // 保留原始平台消息，用于特殊处理
}
```

Agent 只消费 `UnifiedMessage`，也只产出统一响应结构 `UnifiedResponse`（包含 `text`、`embeds`、`actions` 等字段）。这一步是所有后续工作的基础。

### 2. 编写平台适配器

为 Telegram 和 Discord 各写一个适配器插件，负责：

- **入站转换**：将平台 SDK 的原始消息转换为 `UnifiedMessage`
- **出站转换**：将 `UnifiedResponse` 转换为平台要求的 API 调用参数
- **会话映射**：为每个平台维护一个 `sessionId ↔ 平台会话标识` 的双向映射表，确保多轮对话能正确关联

以 Telegram 适配器为例，处理群聊消息时，生成 `sessionId = telegram:group:chatId` 以区分不同群。Discord 则用 `discord:guild:channel` 或再加线程 ID。

### 3. 构建消息路由器

在 OpenClaw 的事件循环中，引入一个路由器，它监听所有适配器的 `onMessage` 事件，统一调度 Agent 的执行：

```
TG Adapter ─┐
            ├─> Message Router ─> Agent Core
DC Adapter ─┘
```

路由器负责负载均衡（如果后续扩展多实例）、错误隔离（一个平台崩溃不影响另一个）、以及平台优先级的按需调整。简单场景下，一个异步队列+单 Agent 实例就足够。

### 4. 处理 Discord 的交互式命令

Discord 的斜杠命令和按钮交互走的是 Interaction 流，必须先用 `deferReply` 或直接回复一个 `Pong`/`Message` 来避免超时。适配器需要在收到 Interaction 时立即返回一个“占位”响应，然后将转化后的 `UnifiedMessage` 推入路由队列。Agent 产生的最终响应通过 `editReply` 或 `followUp` 发送。

### 5. Agent 核心与工具调用

Agent 本身使用 OpenClaw 的标准 API 构建，无需关心消息来源。工具（例如 MCP 提供的搜索、文件操作）仍然正常调用。唯一需要注意的是，某些工具的输出可能包含平台不兼容的内容（例如表格、长代码块），这可以在响应适配阶段由平台适配器进行裁剪或分段。

## 踩坑记录

- **Telegram 消息分段**：TG 消息上限为 4096 字符，MarkdownV2 模式下某些特殊字符需要转义。我写了一个简单的“文本切片器”，在适配器出站时将长文本切分为多条消息连续发送，同时保持代码块闭合。
- **Discord 按钮交互过期**：按钮的 `customId` 携带状态，但 Interaction 令牌只有 15 分钟有效期。如果 Agent 处理耗时过长（如等待外部 API），会导致后续 `editReply` 失败。解决方法是让 Agent 在 15 分钟内至少返回一次中间提示，或者将长时间任务转为 `followUp` 消息。
- **会话 ID 冲突**：最初直接用平台原生 ID 做 sessionId，导致两个平台同用户 ID 碰撞。必须加上平台前缀，并避免使用可能重复的简短数字 ID。
- **Discord 全局速率限制**：在短时间内大量发送消息时，Discord 会返回 429。适配器需要内置退避重试机制，否则容易丢失消息。推荐使用库自带的 rate limit 处理或手动实现指数退避。
- **线程/并发**：Node.js 下同时处理两个平台的事件循环需要注意异步异常捕获。每个适配器各自使用 try/catch 包裹事件处理，防止单个平台异常导致整个 Agent 进程退出。

## 可复用建议

1. **适配器模式**：为每个平台创建独立插件，实现统一的 `IMessageAdapter` 接口。这样新增 WhatsApp、Slack 等渠道时只需编写新适配器，Agent 核心零改动。
2. **中间件管道**：在路由器与 Agent 之间插入中间件，例如日志记录、敏感词过滤、权限校验等，可以同时作用于所有平台。
3. **配置外置**：Token、Webhook URL、允许访问的频道 ID 白名单等都放在环境变量或配置文件中，避免硬编码。使用 OpenClaw 的配置管理能力，为不同平台设置不同行为开关。
4. **平台特性组件化**：将例如 Telegram 的 inline keyboard、Discord 的 Embed 等平台特有的交互抽象为 `Action` 对象，在统一响应中声明，由适配器决定是否渲染并实现。Agent 不必知道底层实现。
5. **监控**：为每条消息添加 trace id，记录从入站到出站的完整链路。当出现丢消息或响应错乱时，可以快速定位是哪个适配器或路由器环节出现了问题。

## 总结

跨平台消息路由的本质，是将“对话智能”与“消息传输”解耦。OpenClaw 的插件机制天然适合这种架构：Agent 作为单一大脑，专注于理解与推理；Telegram 和 Discord 适配器作为外设，负责平台相关的 I/O 转换。按上述步骤落地后，就可以用一个 Agent 实例同时稳定服务两个社区，后续再扩展更多平台也并不困难。

真正投入生产时，多留一份心给各个平台的速率限制与交互超时，往往是决定体验好坏的关键。

---

