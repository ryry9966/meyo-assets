---
title: 一个 Agent 同时接住 Telegram 和 Discord：统一消息路由实践
feedId: 34628
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

很多自动化团队早期会按平台分别维护机器人：Telegram 用一套库，Discord 用另一套。这样做一开始比较直接，但当 Agent 开始处理任务、维护会话状态、调用同一批 MCP 工具时，重复实现会迅速放大。两份命令解析、两套权限、两个回话存储，甚至两个不同版本的 agent 逻辑。

对 Agent 来说，Telegram 和 Discord 用户本质上都只是“一个发来消息、期待一次回复”的调用方。平台差异应该尽量沉到底层适配器，而不是污染核心业务逻辑。

## 问题

核心不是“怎么接入两个平台”，而是怎么让一个 Agent 同时服务两个平台，且互不干扰。具体包括：

- 消息来源标识：同一条消息必须知道来自哪个平台、哪个频道、哪个用户。
- 命令解析差异：Telegram 有 `/cmd@bot` 和实体偏移；Discord 通常靠前缀。
- 回复目标正确：不能把 Discord 的回复发到 Telegram。
- 平台限制不同：消息长度、限流、富文本格式、权限模型都不同。
- 失败隔离：一个平台限流或崩溃，不应影响另一个平台。

## 做法

第一步，定义统一消息信封。所有平台 adapter 都输出同一个结构：

```python
@dataclass
class InboundMessage:
    platform: str
    chat_id: str
    user_id: str
    text: str
    reply_to: str | None = None
    raw: dict = field(default_factory=dict)
```

第二步，适配器只做“翻译”。Telegram adapter 把 Update 转成 `InboundMessage`，Discord adapter 把 Message 转成同一结构。路由层不直接依赖任何平台 SDK。

第三步，路由规则与 Agent 解耦。先归一化文本，再做意图判断：

```python
def normalize(msg: InboundMessage):
    if msg.platform == "telegram":
        text = strip_bot_mention(msg.text)
        if text.startswith("/"):
            return ("command", text[1:].split()[0], text)
    elif msg.platform == "discord":
        if msg.text.startswith("!"):
            return ("command", msg.text[1:].split()[0], msg.text)
    return ("chat", None, msg.text)
```

第四步，Agent 只接收归一化后的 intent 和用户/会话上下文。返回结果时带上原始平台的 `(platform, chat_id)` 回执键，由对应 adapter 发回。这样核心 Agent 不需要知道 Telegram 和 Discord 的存在。

第五步，错误处理也要按平台隔离。建议为每个平台分配独立发送队列，避免共享队列背压。例如 Telegram 被限流时，Discord 的正常回复仍能出队。

## 踩坑点

1. **Telegram 命令实体偏移**：如果拿 Python 字符串直接切命令，遇到 `/cmd@your_bot` 或多字节字符会出错。应使用 Bot API 返回的 `entities`，并注意 UTF-16 偏移。
2. **Discord Message Content Intent**：如果接得到消息但收不到正文，先检查开发者后台是否开启 Message Content Intent，并在代码中声明对应 intent。
3. **消息长度不对称**：Telegram 文本上限约 4096 字符，Discord 约 2000。Agent 输出要按平台分片，而不是统一截断。Telegram 可发多段，Discord 可考虑嵌入或附件。
4. **身份映射**：Telegram user_id 和 Discord user_id 是两套数字，不能直接作为会话主键。如果同一自然人跨平台使用，需要单独建 `account` 或 `binding` 表，否则两个平台的记忆会割裂。
5. **限流与重试**：Discord 对同频道消息速率限制严格，盲目重试会加剧限流。Telegram 的 `429` 响应会带 `retry_after`，应读取并休眠，而不是固定间隔重试。
6. **富文本降级**：Telegram 支持 HTML/Markdown 子集，Discord 有自己格式。Agent 输出如果带格式标签，必须由 adapter 做转换或降为纯文本，不能直接透传。

## 可复用建议

- 路由规则配置化。命令前缀、白名单、黑名单都放到配置或数据库，不要写死在 agent 逻辑里。
- 每条消息进入路由时生成 `trace_id`，日志里统一带 `platform / channel / user_id / trace_id`。排障时能快速定位是 adapter 问题还是 agent 问题。
- 平台能力做 feature flag。例如 Telegram 支持 inline keyboard，Discord 支持 slash command，但可以先禁用差异能力，保持最小可用。
- 为每个平台设置独立超时和重试策略。Telegram 的 webhook 超时和 Discord 的 interaction 超时模型不一样，不要共享。
- 给 Agent 的 system prompt 里不要出现平台专有指令，例如“当用户发 !命令时”。平台差异全部放在适配层。

## 总结

一个 Agent 服务多平台并不需要把平台逻辑塞进 Agent。把平台差异压缩到适配器，用统一消息信封隔离，路由层只处理意图，核心 Agent 保持平台无关。真正花时间的往往不是接入，而是身份映射、限流隔离和消息长度这些工程细节。把这些处理干净，多平台就从负担变成普通能力扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2145f6363856e014.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/594e01abb6364d5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/40c641070f840043.png)

