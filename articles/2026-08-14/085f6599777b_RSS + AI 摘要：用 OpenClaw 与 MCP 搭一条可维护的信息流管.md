---
title: RSS + AI 摘要：用 OpenClaw 与 MCP 搭一条可维护的信息流管线
feedId: 33085
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

信息源越来越多，但真正值得精读的很少。RSS 能把分散的更新拉回同一条管道，AI 摘要可以进一步把长文压缩成可扫描的要点。问题在于：如果只是“抓下来丢给大模型再推送”，这条管线很快就会变成不可维护的玩具——去重混乱、摘要幻觉、成本失控、与 Agent 工具链脱节。

这篇文章记录我在 OpenClaw 环境里用 MCP 暴露摘要能力、用 SQLite 管理状态、用定时任务驱动抓取的工程化做法。目标是：跑得稳、能排障、可复用到不同订阅源。

## 问题拆分

实际要解决的不是“如何调用一次 AI”，而是这四个点：

1. RSS 源格式参差，更新时间、GUID、内容字段经常缺失。
2. 摘要不能只是“看起来有道理”，需要能回到原文核对。
3. 管线需要被 Agent 按需查询，而不是单向推送刷屏。
4. 抓取、去重、摘要、推送各阶段要有清晰状态，出错可重试。

## 做法/步骤

### 1. 统一数据结构

我用 Python 的 `feedparser` 抓取，只保留必要字段，避免直接把源里的杂散字段塞进存储。

```python
item = {
    "feed_id": feed_id,
    "item_id": entry.get("id") or entry.get("link"),
    "link_hash": sha256(entry.link.encode()).hexdigest(),
    "title": entry.title,
    "content": clean_html(entry.summary or entry.get("content", [{}])[0].get("value", "")),
    "published_at": entry.get("published_parsed") or entry.get("updated_parsed"),
    "status": "pending",
}
```

`item_id` 不一定可靠，所以额外用 `link_hash` 作为去重主键。SQLite 表里对 `link_hash` 建唯一索引。

### 2. 状态流转

状态只有四类：`pending`、`summarized`、`failed`、`skipped`。这样每次运行可以只处理 `pending`，失败项单独重试，不会重复消耗模型额度。

```sql
CREATE TABLE items (
  id INTEGER PRIMARY KEY,
  feed_id TEXT NOT NULL,
  link_hash TEXT NOT NULL UNIQUE,
  title TEXT,
  content TEXT,
  published_at TEXT,
  status TEXT DEFAULT 'pending',
  summary TEXT,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### 3. AI 摘要用固定 Prompt 与 JSON 输出

摘要阶段只做一件事：把原文压缩成便于 Agent 使用的结构化字段。Prompt 固定为：

```text
你是信息摘要助手。基于以下文章内容输出 JSON：
{
  "core_points": ["不超过5条"],
  "relevance": "high|medium|low",
  "unverified_claims": ["原文中未证实或可能存在争议的内容"],
  "source_bias": "neutral|opinion|promotional"
}
只依据原文，不要补充外部知识；无法判断时写“原文未说明”。
```

输入内容会先做 HTML 清洗和截断，单篇最多 4000 token，避免长文把上下文撑爆。输出用 JSON schema 校验，不合法就重试一次，仍失败则标记 `failed`。

### 4. 通过 MCP 暴露给 OpenClaw Agent

摘要结果不直接推送，而是作为 MCP server 的工具让 OpenClaw Agent 按需调用。我定义了三个工具：

- `list_summaries(status="unread", limit=20)`：分页读取摘要。
- `get_item(id)`：获取单条原文与摘要，便于回查。
- `mark_read(id)`：标记已处理。

这样 OpenClaw 可以在用户提问时检索“最近有哪些高相关度信息”，而不是被通知淹没。MCP server 用标准 JSON-RPC 方式暴露，配置文件里指向同一 SQLite 数据库，Agent 侧无需关心抓取细节。

### 5. 定时抓取与推送

抓取任务用系统 cron 每 30 分钟跑一次。每轮限制单个源最多处理 10 条，抓取后立即写入 SQLite，状态为 `pending`。摘要任务只处理 `pending`，成功后状态改为 `summarized`。推送只针对高相关度且用户配置了即时通知的源，避免逐条打扰。

## 踩坑点

- **RSS 更新时间不可信**：很多源没有 `updated`，或把更新时间写成抓取时间。去重不要只依赖时间，用 `link_hash` 更稳。
- **摘要幻觉**：大模型容易“合理补充”原文没有的信息。必须在 Prompt 里要求输出 `unverified_claims`，并保留原文链接供回查。
- **XML 中的 HTML 实体和脚本片段**：清洗不彻底会导致 token 爆炸或摘要被无关内容带偏。建议用 `bleach` 或 `lxml` 保留段落文本，去掉 `script/style`。
- **MCP 工具返回大列表**：不要一次性把几百条摘要塞给 Agent，否则上下文很快耗尽。分页和 `status` 过滤是必须的。
- **成本控制**：批处理时限制并发，失败使用指数退避；设置单日最大摘要条数，超出直接标记 `skipped`。

## 可复用建议

- 先跑通 2-3 个稳定源，验证去重和状态流转，再扩展订阅列表。
- Prompt 和 JSON schema 单独放在文件里，与代码一起做版本管理。改 Prompt 等于改管线行为。
- 记录关键指标：每轮抓取数、去重命中数、摘要成功/失败数、单篇平均 token。没有这些指标，出问题时只能盲猜。
- 如果 OpenClaw 需要更主动的交互，可以把“订阅源管理”也做成 MCP 工具，例如 `add_feed`、`remove_feed`、`list_feeds`，让 Agent 在对话中调整源。

## 总结

RSS + AI 摘要的难点不在“能不能生成摘要”，而在如何把抓取、去重、摘要、查询做成一条有状态、可重试、可观测的管线。用 SQLite 做状态中心，用固定 Prompt 控制摘要质量，用 MCP 把结果暴露给 OpenClaw Agent，这条管线才能从一次性脚本变成日常可依赖的信息基础设施。

---

