---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 33804
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

很多 Agent 项目最初只在一个 IM 平台落地，通常先接 Telegram。等需要服务 Discord 社区时，如果直接复制一套 bot 逻辑，很快会遇到三类问题：

- 指令行为不一致，两个平台的用户得到不同体验；
- 会话状态分裂，排查问题要在两套日志里来回跳；
- 每次改 prompt、工具链或流程，都要改两份代码。

这篇文章记录一种工程化做法：把 Agent 核心与平台解耦，让 Telegram 和 Discord 都作为可插拔适配器接入同一个消息路由。

## 问题

双平台接入并不是“把两个 SDK 塞进同一个 handler”。实际差异比想象中大：

- 消息长度限制：Telegram 约 4096 字符，Discord 为 2000 字符；
- Markdown 方言不同：Telegram MarkdownV2 转义规则严格，Discord 只支持子集；
- 身份标识不同：Telegram 用户 ID 是数字，Discord 用户 ID 是 snowflake；
- 交互模式不同：Discord slash command 和 Telegram callback query 的生命周期不一样；
- 限流与鉴权：Telegram webhook secret token，Discord Ed25519 签名验证。

如果让 Agent 核心直接处理这些平台细节，核心逻辑会被污染。

## 做法

### 1. 统一入站消息模型

先定义一个与平台无关的消息结构：

```go
type Inbound struct {
    Platform  string // telegram | discord
    ChatID    string
    UserID    string
    MessageID string
    Text      string
    ReplyTo   string
    Timestamp time.Time
    Raw       any // 平台原始事件
}
```

附件不直接塞进结构体，统一上传到对象存储或本地目录，只保留引用。

### 2. 平台适配器负责翻译

Telegram 适配器接收 update，Discord 适配器接收 message 或 interaction，统一转换成 `Inbound`，发到内部队列。

出站侧也抽象成动作接口：

```go
type Outbound interface {
    Reply(ctx context.Context, in Inbound, text string) error
    React(ctx context.Context, in Inbound, emoji string) error
    Edit(ctx context.Context, in Inbound, text string) error
    SendTyping(ctx context.Context, in Inbound) error
}
```

核心 Agent 只调用 `Outbound`，不知道消息最终发到哪个平台。

### 3. 队列与幂等

入站消息进入 Redis list 或本地 channel，由 worker 消费。处理前先做幂等去重，建议以 `platform + message_id` 为 key，TTL 设置为 6–24 小时。

长任务不要阻塞 webhook。可以先 ACK，再异步处理，或者先回一个“处理中”的状态。

### 4. 会话管理

每个平台独立使用 `{platform}:{userId}` 作为会话键。不要隐式跨平台合并，容易串号。需要跨平台打通时，让用户显式执行 `/bind` 指令，绑定关系存 Redis 或数据库。

### 5. 输出转换

核心先用统一 Markdown 生成回复，再由出站适配器转换：

- Telegram 做 MarkdownV2 转义；
- Discord 保留安全子集；
- 超长内容按平台限制分片发送。

## 踩坑点

**Discord interaction 3 秒超时**  
slash command 和按钮交互必须快速响应。如果处理时间较长，先返回 deferred，再通过 webhook 补发结果。

**Telegram callback query 也要及时 answer**  
不 answer 会导致客户端一直转圈，用户以为没反应。

**webhook 鉴权不能省略**  
Telegram 验证 `X-Telegram-Bot-Api-Secret-Token`，Discord 验证 `X-Signature-Ed25519` 和 timestamp。不要只依赖 IP 白名单。

**消息编辑/删除事件重复**  
同一个 `message_id` 可能触发多次 update。只做幂等去重还不够，必要时根据事件类型过滤，比如只处理新消息和明确需要响应的回调。

**限流**  
Telegram 单 chat 建议控制在 1 msg/s 左右，Discord 全局约 50 requests/s。出站适配器加 token bucket，遇到 429 做指数退避。

**日志要带 trace id 和平台标识**  
双平台用户 ID 混在一起非常难排查。建议每条日志都带 `platform` 和一条从入站到出站贯穿的 `trace_id`。

## 可复用建议

- 保持核心 Agent 与平台无关，新增平台只写 adapter；
- 所有平台消息进入同一条日志流，调试时能看到完整链路；
- 用 fake outbound 做核心单测，平台适配器只做薄集成测试；
- 平台失败要隔离：Telegram token 失效不应影响 Discord 服务；
- 目录结构建议清晰分层：

```text
internal/
  core/       # Agent 核心
  envelope/   # 统一消息模型
  adapters/
    telegram/
    discord/
  outbound/   # 出站动作接口
```

## 总结

跨平台消息路由的关键不是同时接入两个 SDK，而是把平台差异封印在适配层，让 Agent 只处理统一语义。这样双平台行为一致，后续扩展 Slack、Matrix 或 Web 端也会更顺。

工程上先做统一模型，再做队列和幂等，最后处理平台限流与交互细节。不要一上来就写两个独立的 bot。

---

