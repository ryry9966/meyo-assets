---
title: RSS + AI 摘要：给 OpenClaw 加一条可恢复的自动化信息流管线
feedId: 33061
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

RSS 没有死，但原始订阅源的噪音确实很高。很多技术源每天更新几十条，真正值得读的不到五分之一。用 AI 摘要做一层过滤，可以只把高价值条目推到眼前。问题在于：抓取、去重、摘要、推送这条链路很容易写成一次性脚本，跑几天就出现重复推送、源失效、token 成本失控。

这篇文章面向 OpenClaw/Agent/MCP/自动化实践用户，讲一条可恢复、可观测的 RSS + AI 摘要管线怎么搭。

## 问题拆解

稳定管线要解决四件事：

1. 抓取失败与源失效
2. 条目去重，避免重复摘要
3. 摘要质量与 token 成本
4. 推送可靠，失败可重发

下面按步骤拆开。

## 做法/步骤

### 1. 抓取与解析

用 Python 的 `feedparser`，比手写 XML 解析省心。每个源单独配置：

```python
FEEDS = [
    {"name": "hn", "url": "https://hnrss.org/frontpage", "tags": ["tech"]},
    {"name": "arxiv_cs_ai", "url": "https://arxiv.org/rss/cs.AI", "tags": ["paper"]},
]
```

解析后统一字段：`id`、`title`、`link`、`summary`、`published`、`source`。后续所有模块只依赖这套统一结构，不要直接操作源数据。

### 2. 去重与状态

不要只依赖 `entry.id`，有些站会换 id。建议用 `link + title` 的 sha1 作为指纹。状态存 SQLite，记录已处理指纹和摘要时间。

```sql
CREATE TABLE IF NOT EXISTS seen (
  fingerprint TEXT PRIMARY KEY,
  source TEXT,
  title TEXT,
  link TEXT,
  summarized_at TEXT
);
```

每次处理前先查 fingerprint，已存在则跳过。这样即使中途崩溃，也不会重复摘要。

### 3. AI 摘要

把摘要做成一个纯函数：输入是标题、原始摘要、正文片段；输出是 JSON，约束字段。

提示词示例：

```
你是信息过滤助手。用中文输出 JSON，不要解释。
{
  "importance": "high|medium|low",
  "summary": "不超过60字",
  "reason": "为什么值得看/不值得看",
  "action": "read|skip"
}
```

只对 `importance` 为 `high` 或 `medium` 的条目推送。这样能控制推送噪音和 token 消耗。使用 OpenAI-compatible API，设置 `max_tokens: 200`、`temperature: 0.2`，输出稳定。

### 4. 在 OpenClaw 中编排

不建议自己写常驻 cron。把“抓取 + 去重 + 摘要 + 推送”封装成 MCP server，暴露两个工具：

- `fetch_rss(source: str) -> list[Entry]`
- `summarize_entries(entries: list) -> list[Digest]`

然后在 OpenClaw 里配一个定时 Agent，每 30 分钟调一次。OpenClaw 负责调度、错误回传和日志。MCP 工具比硬编码插件更利于复用，之后接入其他 Agent 也直接可用。

### 5. 推送

推送渠道单独拆开：Telegram Bot、Slack Webhook、ntfy 都可以。失败时把待推送内容写回 SQLite，下一次运行重试。不要边摘要边推送，容易丢状态。

## 踩坑点

- **Cloudflare 403**：很多 RSS 源会拦默认 UA。设置 `User-Agent` 为正常浏览器 UA，必要时加条件请求。
- **entry.id 不一致**：部分源每次抓取都换 id，必须用 `link+title` 指纹。
- **摘要上下文溢出**：不要塞全文，截取前 1500 字符，否则 token 爆炸。
- **时区混乱**：`published` 可能是 UTC 或带偏移，统一转成 UTC 再入库，避免重复处理。
- **推送幂等**：Telegram Bot 发送失败后不要清状态，只有 `push_success=1` 才标记完成。
- **MCP 工具超时**：批量摘要时单次不要超过 10 条，否则 LLM 响应可能超过 MCP 端限制。

## 可复用建议

- 源列表外置到 YAML/JSON，别硬编码。
- 状态库用 SQLite，比 pickle 稳，还能 SQL 排查。
- 分阶段日志：抓取、去重、摘要、推送每阶段都打条数，问题一眼定位。
- 先跑 3 个源灰度一周，再逐步加源。
- 为每个源设置 `enabled` 开关和 `last_error` 字段，源失效自动停用。
- 摘要结果落盘，后续可用于全文检索或做个人知识库。

## 总结

RSS + AI 摘要的工程难点不在“能不能跑通”，而在“跑一个月后还准不准、丢不丢、贵不贵”。把边界切清楚：抓取只管抓取，去重只看去重，摘要只管摘要，推送只管推送。用 OpenClaw/MCP 做调度，用 SQLite 做状态，就能得到一条可恢复、可观测、成本可控的信息流管线。

---

