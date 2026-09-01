---
title: RSS + AI 摘要：搭建一条少打扰、可回看的自动化信息流管线
feedId: 35642
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

我订阅的 RSS 源从十几个涨到六十多个，每天新增条目上百条，真正值得读的不到十分之一。试过一些现成摘要服务，要么摘要质量不稳定，要么无法按自己的规则过滤，更没办法接进现有 Agent 工作流。后来决定自己搭一条轻量管线：定时抓取 RSS → 清洗去重 → AI 摘要与评分 → 汇总推送/落库。这条管线跑了一阵子，基本稳定，也踩了一些坑。

## 问题

核心问题不是“能不能摘要”，而是如何让摘要结果可信、成本可控、并能被 OpenClaw 或 MCP 这类工具消费。

直接对每篇全文做摘要不现实：很多 RSS 只提供摘要，需要额外抓原文；全量调用大模型成本高、延迟大；不同源格式差异大，去重和状态管理容易出错。更关键的是，摘要输出如果不结构化，后续自动化很难用。

所以目标拆成几件事：

- 稳定抓取：支持常见 RSS/Atom，处理编码、相对链接、缺少 guid 等情况。
- 轻量清洗：提取标题、链接、发布时间、正文或摘要，截断到合理长度。
- 结构化 AI 输出：让模型返回 JSON，包含一句话摘要、关键点、相关度评分、建议动作。
- 分发与集成：生成日报，推送通知，并暴露成 MCP 工具或命令行，供 OpenClaw 查询。

## 做法/步骤

我用的栈很普通：Python + feedparser + SQLite + httpx + OpenAI 兼容接口。整个管线可以拆成几个脚本，也可以做成一个包。

### 1. 抓取与状态管理

用 feedparser 拉取每个源，解析 entries。为避免重复，每个条目需要一个稳定 ID。优先使用 entry.id，如果没有就用 link 做 SHA-1。SQLite 里建一张 `seen_items` 表，记录 `source_id, item_hash, first_seen_at, last_updated_at`。抓取时先查是否已存在，存在就跳过。

同时利用 HTTP 层缓存：保存每个源的 ETag 和 Last-Modified，下次请求带上。很多源支持 304，能减少抓取压力。

### 2. 清洗与正文获取

大多数条目只有 summary，直接给 AI 摘要可能信息不足。我的做法是：

- 先尝试从 summary/description 提取纯文本，去掉 HTML 标签。
- 如果 summary 长度小于 100 字符，或者内容明显是“点击查看全文”，就用 trafilatura 抓原文。
- 原文抓取后同样提取正文，限制最大 2000 字符左右，避免 token 浪费。

这一步失败率不低，需要容错：抓取失败就退回 summary，再失败就标记为 `skip`，不阻塞整个流程。

### 3. AI 摘要结构化输出

这是核心。为了稳定拿到结构化结果，我用 JSON mode（或 function calling）。Prompt 大致是：

```text
You are a personal information filter. Given the title and content of an article, return JSON with fields:
- summary: one sentence in Chinese
- key_points: list of up to 3 short points
- relevance_score: 0-10, how relevant to topics like AI agents, MCP, automation, open source tools
- action: "read" | "skim" | "skip"
```

请求时设置 `response_format={"type": "json_object"}`，并校验返回的 JSON 字段完整性。如果解析失败，重试一次；再失败就丢弃该条目并记录日志，不手动干预。

模型选择上，初筛用便宜的模型（如 GPT-4o mini 或 Claude Haiku），只有 `action=read` 且 `relevance_score>=7` 的条目，才在需要时用更强的模型做深度摘要。我在日报里只推送高相关度的条目，其余存入数据库备查。

### 4. 分发与集成

每天定时运行一次（或每 6 小时），生成 Markdown 日报：

- 高相关度条目：标题、一句话摘要、关键点、原文链接。
- 统计信息：新增条目数、跳过数、抓取失败源列表。

推送渠道可以根据自己的习惯：Telegram bot、企业微信、邮件，或者直接写入本地 Markdown 文件。我选择写入一个 `daily-digest` 目录，同时通过一个简单的 MCP server 暴露查询接口，这样在 OpenClaw 里可以直接问“今天有哪些值得读的 AI 工具文章？”。

简单 MCP 工具可以这样设计：

```json
{
  "name": "query_daily_digest",
  "description": "Query today's RSS AI digest by relevance score or keyword",
  "parameters": {
    "date": "string, optional, default today",
    "min_score": "number, optional, default 7"
  }
}
```

这样 OpenClaw 的 Agent 能按需检索，而不只是被动收推送。

## 踩坑点

1. **RSS 源不规范**：有些源没 guid，有些 guid 不稳定（比如每次抓取都变）。用 link 做 hash 比 guid 更可靠，但要注意同一个链接可能带不同 utm 参数，需规范化 URL。
2. **时间比较**：不要把本地时间直接写库，统一存 UTC，展示时再转本地。cron 任务也容易受服务器时区影响，建议在容器里设 `TZ=UTC`，日报生成时再转。
3. **原文抓取频率**：不要对同一域名频繁请求，加随机延迟，尊重 robots。trafilatura 对某些站点会失败，需要 fallback。
4. **AI 限流与重试**：批量请求时用 asyncio + semaphore 控制并发，遇到 429 或 5xx 指数退避。如果某天摘要量特别大，控制每天最多处理 200 条，避免成本失控。
5. **去重**：跨源重复内容去重比较麻烦。可以用标题相似度（如 difflib）做二次去重，但误杀率较高。我目前只做同源去重，跨源重复靠人工在日报里忽略。

## 可复用建议

- 把抓取、清洗、摘要、分发拆成独立函数，方便替换 AI provider 或推送渠道。
- 摘要结果一定要落库，保留原文链接和原始摘要，方便回溯。
- 日志要细：记录每个源的抓取状态、条目数、失败原因。目前我用简单的 logging + SQLite 表，够用。
- 如果你想接入 OpenClaw，建议先做成 MCP server，而不是只写一个脚本。MCP 的接口天然适合 Agent 调用。
- 不要把管线做成“全自动无人值守”后就放手，至少每周看一眼失败源列表，很多 RSS 源会悄悄改版或失效。

## 总结

这条 RSS + AI 摘要管线没有复杂的技术门槛，核心价值在于把“刷信息流”变成“处理结构化摘要 + 按需深读”。它省下的是注意力和重复决策成本。对 OpenClaw 社区来说，更有意义的是把它作为一个可被 Agent 调用的信息入口：抓取和摘要负责降噪，Agent 负责根据上下文做进一步筛选和问答。

如果你也在用 OpenClaw 或 MCP，可以试着把日报查询做成一个工具，后续还能扩展成让 Agent 自动标记、归档甚至反向订阅新源。几个脚本跑通后，它就会成为你自动化工作流里很稳定的一小块。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1c88c46d21a43150.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/6799840b22285dd7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/ae3783afa6b4e5af.png)

