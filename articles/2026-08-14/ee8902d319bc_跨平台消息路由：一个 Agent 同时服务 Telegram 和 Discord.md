---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 33015
source: 综合讨论
publishedAt: 2026-08-14
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

不少自建 Agent 的路径是先在 Telegram 上跑通，然后想接入 Discord。初期通常直接在 bot 回调里写业务逻辑，Telegram 的 `message` 和 Discord 的 `interaction` 结构不同，很快就出现 `if platform == "telegram"` 之类的分支。平台差异不只是消息格式，还包括 Markdown、按钮回调、编辑、延迟响应、速率限制等。继续在 Agent 核心里堆平台判断，最后会变成很难维护的胶水代码。

这篇文章不讨论某个具体 Agent 框架的内部实现，只整理一个可复用的路由层做法。无论你用的是 OpenClaw、其他 Agent runtime，还是自己写的调度器，思路都类似。

## 问题

Telegram 和 Discord 至少有几类差异：

- 消息通道标识不同：TG 是 `chat_id`，Discord 是 `channel_id`/`guild_id`。
- 文本格式不同：TG 支持 MarkdownV2/HTML，Discord 支持受限 Markdown。
- Discord 的 slash command/button 交互通常要求 3 秒内 ACK，否则用户端会显示失败。
- Telegram webhook 如果同步处理太久，会触发超时重试。
- 消息长度、文件上传、编辑/删除能力、私聊与群聊权限模型也不同。

如果直接在 Agent 核心逻辑里写平台判断，业务代码会被平台差异污染；一旦加第三个平台，成本会继续上升。

## 做法/步骤

### 1. 先定义统一入站 envelope

让所有平台回调先被适配器转换成同一种结构：

```text
{
  id: string,
  platform: "telegram" | "discord",
  chat_id: string,
  thread_id?: string,
  sender_id: string,
  text: string,
  type: "text" | "command" | "callback" | "mention",
  raw: object,
  reply_to_id?: string,
  ts: number
}
```

`raw` 保留平台原始 payload，方便处理平台特有字段。不要为了统一把所有信息强行塞进固定字段，否则后期会很痛苦。

### 2. 定义统一出站 action

Agent 不直接调用平台 API，而是返回抽象动作：

```text
send_text / edit_text / send_embed / set_buttons / react / delete
```

由平台适配器实现这些动作。例如 Telegram 的 `send_text` 需要处理 `chat_id` 和 `parse_mode`，Discord 的 `send_text` 可能要决定是回复 webhook 还是发送 channel message。

### 3. Agent 只依赖统一接口

核心逻辑只处理业务：收到什么、该回什么、要不要调用工具。平台适配器作为 channel 插件注册 start/stop/send 能力。如果你已经有 MCP/插件体系，建议把平台适配器做成 channel 插件，而不是把每个平台事件暴露成动态 tool。动态 tool 适合工具调用，不适合承载平台生命周期和回执管理。

### 4. 会话与路由

会话键建议用 `(platform, chat_id, thread_id?)`。同一个用户从 TG 和 Discord 发消息，不要默认合并到同一上下文，否则容易出现跨平台串线。如果确实需要身份合并，再显式做 user mapping。

### 5. 响应策略

入站后先快速 ACK，再处理耗时任务。Telegram webhook 场景下先返回 200，把任务丢进队列；Discord 交互场景下先 `defer` 或回复“处理中”，再异步更新结果。

## 踩坑点

- **Discord 3 秒 ACK**：slash command/button 处理必须快速响应。不要在交互回调里做模型推理。可以先 `deferUpdate` 或 `deferReply`，完成后用 `editReply` 更新。
- **Telegram webhook 超时**：TG 会等待 HTTP 响应，长时间任务同步处理会导致超时重试，用户收到重复回复。建议 webhook 只做验签和入队，业务处理在 worker 中完成。
- **Markdown 差异**：不要把 Discord 文本直接发到 TG，也不要把 TG 的 MarkdownV2 发到 Discord。内部尽量用纯文本或结构化内容，由适配器按平台规则转义。
- **长度限制**：TG 单条消息约 4096 字符，Discord 单条约 2000。超长内容需要分片或生成文件。分片逻辑最好放在适配器层，而不是 Agent 核心。
- **编辑与删除**：不是所有平台消息都能编辑。TG 可以编辑 bot 自己发的消息，Discord 可以编辑 webhook/interaction 响应，但能力并不完全一致。适配器应明确返回“是否支持编辑”，Agent 再决定降级为发新消息。
- **权限和隐私模式**：TG 群组里 bot 默认隐私模式可能收不到非 @ 消息；Discord 则需要正确的 intents 和角色权限。接入前先确认事件是否真正到达。
- **消息去重**：按钮回调和用户文本可能几乎同时到达，或同一事件因重试重复投递。用 envelope `id` 做幂等，适配器入口先去重。

## 可复用建议

- **配置驱动渠道**：用 yaml/env 管理平台 token、允许的 chat/channel、功能开关，不要把这些写死在代码里。
- **出站重试与降级**：发送失败时进入 failure channel 或至少记录完整上下文，方便补发，而不是静默丢失。
- **结构化日志**：每条消息带 `trace_id`，从入站到出站可追踪。平台 webhook 很容易出现“消息进来了但没回”，没有 trace 很难排查。
- **健康检查**：Telegram 可看 `getWebhookInfo`，Discord 关注 gateway heartbeat/resume。适配器启动时主动检查，避免 bot 跑了半天其实已经掉线。
- **用 fixture 测试适配器**：录制真实平台 payload 作为 fixture，测试解析和格式化逻辑，不要每次改动都依赖真实平台回归。

## 总结

跨平台消息路由的重点不是“让一个 bot 同时在线”，而是把 Agent 与平台解耦。统一 envelope + 出站 action + 平台适配器，就能用很小的成本维护两个甚至更多平台。不要在 Agent 核心里写平台 if-else，这比多写几个适配器更费劲。

---

