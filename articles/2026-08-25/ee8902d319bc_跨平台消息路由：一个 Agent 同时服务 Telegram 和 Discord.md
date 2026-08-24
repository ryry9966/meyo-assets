---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 34571
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景
不少 OpenClaw 实践会先把 Agent 接到 Telegram 或 Discord 单个平台。但真实用户通常分散在两边：TG 偏向命令式交互，Discord 偏向频道讨论。若为每个平台部署独立 Agent，会出现状态割裂、指令逻辑重复、上下文无法共享的问题。更实际的做法是让一个 Agent core 同时服务多个平台，平台只作为消息进出通道。

## 问题
直接在一个进程里同时接两个平台 SDK，容易出现几种耦合：
- 消息结构不同：TG 用 `message_id`/`chat_id`，Discord 用 `channel_id`/`guild_id`，线程、引用、附件语义也不一致。
- 发送限制不同：TG 约 1 条/秒/会话，Discord 有 5 条/5 秒限制，429 处理不能混用。
- 事件语义不同：编辑、删除、回复、slash command 在两个平台差异很大。
- 如果 core 直接依赖平台 SDK，后续接入 Slack/Matrix 成本会很高。

## 做法
核心思路：把“平台适配器”和“Agent 逻辑”分开，中间用统一 Envelope 路由。

### 1. 统一入站 Envelope
所有平台事件都归一化成同一种结构，再交给 core：

```
type Envelope struct {
    Platform    string
    ChatID      string
    UserID      string
    MessageID   string
    ParentID    string
    Text        string
    Attachments []Attachment
    Raw         json.RawMessage
}
```

适配器只做一件事：把 TG/Discord 事件翻译成 Envelope。core 不 import 任何平台 SDK。

### 2. 统一出站 Reply
core 处理后只返回抽象 Reply：

```
type Reply struct {
    Text     string
    ReplyTo  string
    EditID   string
    DeleteID string
}
```

出站路由根据 `Platform` 选择对应适配器，由适配器序列化为平台格式。若 Reply 带 `EditID`，适配器调用编辑接口；带 `DeleteID` 则删除消息。

### 3. 维护消息 ID 映射
很多场景需要后续编辑/删除自己的回复。建议在适配器层维护 `internal_key -> platform_message_id` 映射。例如 key 可以是 `tg:chat_id:reply_to` 或自生成 UUID。这样 core 无需知道平台消息 ID。

### 4. 出站队列与限流
不要从 core 直接同步发送。每个平台适配器内部维护独立队列：
- TG：限制单聊 1 msg/s，群聊 1 msg/s，广播时需谨慎。
- Discord：控制 5/5s，遇到 429 用 `Retry-After` 做指数退避。
队列最好带重试、死信和日志，避免一次限流阻塞整个 Agent。

### 5. 入站去重
Webhook 重试、服务重启、长轮询 offset 处理不当都可能重复消费。用 `platform + message_id` 做幂等键，Redis/本地 LRU 均可。

## 踩坑点
- **会话键冲突**：不要只拿 `user_id` 当会话 ID。TG 群聊和私聊会混，Discord 还有 server/channel/thread 维度。建议统一用 `platform:chat_id:thread_id` 作为会话键。
- **文本分段**：Discord 单条 2000 字符，TG 单条 4096 字符。长回复分段时注意保留 `reply_to`，避免上下文断裂。
- **命令体系不同**：TG 依靠 BotFather 设置命令，Discord 依赖 slash command 注册。不要共用一套命令解析，适配器层应剥离平台特有的 mention/prefix。
- **编辑与删除**：不是所有消息都能编辑/删除，尤其是自己权限不足或消息过期。适配器需捕获平台错误并降级为“重发一条”或忽略。
- **安全校验**：TG 建议使用 `secret_token` 验证 webhook；Discord 校验签名头。Token 不要打进镜像或前端配置。

## 可复用建议
- 把适配器做成插件/MCP 服务，比如 `tg-adapter` 和 `dc-adapter`，暴露 `/ingest` 和 `/send`，core 通过 HTTP/gRPC 调用。这样切换部署形态更方便。
- 将限流、重试、幂等放在统一的出站网关，而不是各适配器里复制逻辑。
- 为每个平台记录指标：入站延迟、出站成功率、429 次数、重试耗时。没有指标，跨平台问题很难排查。
- 用 fixture JSON 固定 TG/Discord 事件样本做归一化测试，避免只测真实环境。
- 新平台接入时，先只支持文本消息和回复，再逐步补编辑、删除、附件、线程。

## 总结
一个 Agent 同时服务 Telegram 和 Discord，本质不是“接两个 SDK”，而是定义清晰的平台边界：core 只处理业务，适配器只处理平台。先把统一 Envelope、出站队列、ID 映射和去重做好，后续加平台只是加适配器。这样既避免状态割裂，也能让同一个 Agent 在不同平台保持一致的上下文和行为。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/71d7ace409fe949c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5599fa00ab57e1cb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e6942ccc2a0074e7.png)

