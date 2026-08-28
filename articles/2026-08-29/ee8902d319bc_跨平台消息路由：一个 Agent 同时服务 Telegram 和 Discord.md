---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 35125
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

很多小团队同时维护 Telegram 和 Discord 两个社群，原本分别跑两个 bot，Agent 逻辑、权限判断、工具调用各写一套。当 Agent 需要调用同一个 MCP 工具并回复到两个平台时，重复代码很容易导致行为不一致：同一个命令在 TG 能跑通，在 Discord 却因为消息格式或权限判断不同而失败。

目标是把 Agent 核心与平台接入解耦：一个核心，多个 adapter。本文记录一个可落地的跨平台消息路由思路，适合已经在用 OpenClaw/Agent/MCP/插件做自动化的实践者。

## 问题

直接在一个进程里接两个 SDK，通常会在消息处理里出现大量 `if platform == "telegram"` 判断。平台差异不只是 API 不同，还包括：

- 消息 ID、用户 ID、会话 ID 体系不同
- 回复、编辑、删除、按钮交互语义不同
- Markdown 子集和转义规则不同
- 限流规则和附件大小限制不同
- 事件回调中超时约束不同

如果核心逻辑与平台 API 耦合，后续接 Slack 或 WhatsApp 会继续膨胀。需要定义统一消息模型，把差异限制在 adapter 层。

## 做法

### 1. 统一入站消息模型

先定义一层与平台无关的结构，例如：

```ts
type InboundMessage = {
  platform: "telegram" | "discord";
  chatId: string;
  userId: string;
  messageId: string;
  text: string;
  attachments: { type: string; url?: string }[];
  replyToMessageId?: string;
  raw: unknown;
};
```

平台 adapter 只负责把 Telegram Update 或 Discord Message 转成这个结构。Agent 核心不 import 任何平台 SDK。

### 2. 统一出站意图

Agent 核心不直接拼 TG/Discord 消息，只产出 Reply 意图：

```ts
type Reply = {
  text: string;
  attachments?: { type: string; url?: string }[];
  actions?: { actionId: string; label: string }[];
  target: { platform: string; chatId: string; replyToMessageId?: string };
};
```

Adapter 再根据平台渲染。比如 Telegram 使用 HTML 或 Markdown v2，Discord 使用 Markdown 或 embed。这样核心逻辑不用关心平台差异。

### 3. 用户绑定与权限

两个平台用户体系不同，如果要做跨平台持久记忆或权限控制，需要绑定表。v1 建议每个平台独立身份，只允许管理员 ID 或白名单触发敏感工具。后续需要跨平台绑定时，可以做 `/bind <token>` 命令，两边都能执行。

### 4. 入站路由与队列

入站事件 → normalize → 入队 → agent core → 出站 adapter。建议至少用 in-process queue 做异步，因为 Discord interaction 和 Telegram callback query 都有 ack 时限。Agent 内部调用 MCP 工具可能超过 3 秒，必须异步响应。

### 5. 配置和部署

每个平台独立配置 token、启用开关、限流参数、管理 ID。一个进程可以跑两个 gateway worker，共享同一个 agent core。Telegram 可选 webhook 或 polling，Discord 一般走 Gateway/WebSocket。生产环境建议用 webhook + 反代，便于灰度和重启不丢事件。

## 踩坑点

- **消息长度**：TG bot 文本上限 4096 字符，Discord 普通消息上限 2000 字符。统一回复如果超长，在出站 adapter 做分片或改发文件，不要依赖某平台自动截断。
- **Markdown 差异**：TG 的 Markdown v2 和 Discord Markdown 转义规则不同，尤其 `_`、`*`、`[`、`]` 容易出问题。建议核心输出纯文本或结构化内容，由 adapter 渲染。
- **按钮交互**：TG 用 `callback_query`，Discord 用 component interaction。统一 `action_id` 后，回调 payload 需要带上 platform、chatId、原消息 ID，否则无法知道事件来自哪个平台。
- **编辑和删除事件**：v1 建议忽略编辑事件，只处理 slash command 和新消息。编辑重放很容易导致 Agent 重复调用 MCP 工具。
- **限流**：TG 全局约 30 msg/sec，Discord 按 route 和 server 有不同限制。批量推送时不要一个循环无脑发，最好在出站 adapter 加令牌桶或队列。
- **幂等**：出站 API 可能超时但实际已发送，重试会导致重复。为每条出站消息生成 idempotency key，或至少在日志里记录 message_id 用于去重。
- **附件大小**：TG bot 文件最大约 50MB，Discord 免费档上传约 8MB。超过阈值上传到对象存储发链接，而不是直接发文件。

## 可复用建议

1. 先做单向命令式 bot，只响应 `/ask` 这类 slash command，不做自然语言监听，减少消息处理分支。
2. 把 adapter 做成插件，平台接入可开关。后续接新平台只需要实现 normalize/render 两个核心方法。
3. 所有入站 raw payload 保留结构化日志，方便排查渲染和权限问题。
4. 出站 adapter 统一健康检查端点，例如 `/health/telegram`、`/health/discord`，暴露重连次数、队列长度、最近错误。
5. 写一个跨平台测试矩阵：文本、Markdown、图片、按钮、回复、编辑、删除。每个平台至少覆盖这些用例。
6. 限流、重试、权限放在 adapter 层，不要在 Agent 核心里散落平台判断。

## 总结

一个 Agent 同时服务多个平台，本质不是把两个 SDK 接进来，而是建立统一消息模型和出站意图，并把平台差异、限流、渲染、权限关进 adapter。这样 Agent 核心和 MCP 工具可以只面向统一结构，跨平台路由的可维护性会明显提升。先做简单命令式接入，稳定后再扩展按钮、编辑同步等复杂能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/d12ee8d076f235f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/93274e6669863c04.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/893bd65fd677bedc.png)

