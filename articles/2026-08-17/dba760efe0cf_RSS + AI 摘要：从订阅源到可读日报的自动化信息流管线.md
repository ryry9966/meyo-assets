---
title: RSS + AI 摘要：从订阅源到可读日报的自动化信息流管线
feedId: 33639
source: 综合讨论
publishedAt: 2026-08-17
---

# RSS + AI 摘要：从订阅源到可读日报的自动化信息流管线

## 背景

RSS 并没有死，它只是退回了工程师和重度信息消费者的工具箱里。相比算法推荐，RSS 的优势在于源可控、无平台噪声、可自由组合。但当订阅源超过几十个，每天新增上百条条目时，人肉筛选已经不可维护。全文太长、标题党太多、重复内容也不少。

在 OpenClaw / Agent / MCP 的实践环境里，RSS 很适合被改造成一条自动化信息流管线：抓取、清洗、去重、AI 摘要、结构化输出，最后推送到 IM 或通过 MCP 工具开放给 Agent 查询。这条管线不依赖特定平台，成本低，且能保持对数据和 prompt 的完全控制。

## 问题

直接“抓 RSS 然后丢给 AI 总结”听起来简单，实际会遇到几类工程问题：

1. **源解析不一致**：不同 feed 的字段差异很大，有的提供 `content:encoded`，有的只有 `summary`；可能是 HTML、纯文本或混合格式。
2. **去重困难**：同一篇文章可能出现在多个 feed，或者同一 feed 因更新时间变化而重复输出；只按标题去重不够可靠。
3. **摘要质量与成本难平衡**：全文太长会消耗大量 token 并增加延迟，截断过多又会丢失关键信息。
4. **调度与容错**：定时任务失败、AI 输出损坏、推送重复，这些都需要幂等和可观测性兜底。
5. **Agent 消费方式**：如果只是每天推一条消息，Agent 无法按需检索。更灵活的做法是把摘要库暴露成 MCP 工具。

## 做法 / 步骤

### 1. 数据采集层：RSS Fetcher

使用稳定、轻量的解析库。Node 环境可用 `rss-parser`，Python 可用 `feedparser`。抓取时保留以下字段：

- `guid` / `id`：作为主去重键
- `link`：原文链接
- `title`
- `content` / `summary`
- `isoDate` / `pubDate`
- `feedUrl` 和 `sourceTitle`：用于标记来源

示例 minimal fetch：

```python
import feedparser

def fetch(url):
    d = feedparser.parse(url)
    entries = []
    for e in d.entries:
        entries.append({
            "guid": e.get("id") or e.get("link"),
            "link": e.get("link"),
            "title": e.get("title", ""),
            "content": e.get("content", [{}])[0].get("value", "") or e.get("summary", ""),
            "published": e.get("published") or e.get("updated"),
        })
    return entries
```

### 2. 清洗与去重

清洗主要做三件事：

- 提取纯文本：用 `html2text` 或 `BeautifulSoup` 去掉 HTML 标签和脚本。
- 内容截断：每篇保留 2000–4000 字符，优先保留开头和包含关键词的段落。
- 生成短 hash：以 `guid` 或 `link` 为准，存 SQLite 或 JSON 文件。对历史 N 天内的 hash 做去重，防止重复推送。

建议保留原始 HTML 内容或至少原文链接，摘要只用于筛选，不替代阅读。

### 3. AI 摘要与结构化输出

批量调用 LLM，建议使用结构化输出能力，强制返回 JSON：

```json
{
  "title": "中文标题或改写标题",
  "summary": "80-120字摘要",
  "tags": ["ai", "rss"],
  "importance": "high | medium | low"
}
```

Prompt 需要明确：

- 不要扩写，不要评价。
- 如果原文信息不足，返回 `importance: low`。
- 保留事实、数字和技术名称。

工程上建议：

- 低 `temperature`，开启 JSON mode 或 function calling。
- 每批限制并发，例如同时 3–5 篇，失败自动重试一次。
- 记录每次调用的耗时、token 使用和解析错误。

### 4. 推送与 Agent 消费

简单场景可以生成 Markdown 日报，通过 Webhook 发到 Slack / 钉钉 / Telegram。

对 OpenClaw / Agent 更友好的做法是增加 MCP server：

- `list_entries(from, to, tags)`：查询指定时间范围内的条目。
- `get_summary(id)`：返回单条摘要与原文链接。
- `generate_digest(date)`：按重要性排序生成当日 digest。

这样 Agent 不用被动接收推送，可以按任务需求主动检索信息。

### 5. 调度

个人使用可跑 `cron`，服务器上可跑 systemd timer，也可以放 GitHub Actions 每日触发。关键是无状态、幂等：每次运行先抓取，再根据 hash 判断哪些是新增，再判断是否需要生成新的摘要。

## 踩坑点

### 1. Feed 解析兼容性

不要假设每个 feed 都规范。有些 feed 用 `dc:date` 而不是 `pubDate`；有些老 blog 的编码不是 UTF-8。解析时统一处理编码，日期优先取 `published` 再退回到 `updated`。

### 2. 去重键不稳定

部分网站会修改 URL 的 query 参数，或同一篇文章在不同 feed 里使用不同 link。仅用 link 去重会漏掉。建议组合 `(normalized_link, title_hash)` 做辅助判断，并将 `guid` 作为主键。

### 3. HTML 清洗不彻底导致 token 浪费

直接截取 HTML 会把大量标签和脚本喂给 LLM。清洗必须在截断之前完成。可先用 CSS selector 或正文提取算法，只保留正文容器内的文本。

### 4. AI 输出解析失败

JSON mode 也不能保证 100% 成功。遇到解析失败时，不要把整个 batch 丢弃。可以降级为保存空摘要并标记为 `needs_review`，或重新调用一次。记录原始错误，便于后续调整 prompt。

### 5. 推送重复

定时任务重试会造成重复推送。可以在发送前写入 `last_pushed_at`，并通过数据库唯一约束保证同一批次只发送一次。对于 IM 推送，可合并为一条消息，减少打扰。

## 可复用建议

- **配置与代码分离**：订阅源、过滤规则、摘要长度、推送目标都放配置文件。新增源不碰代码。
- **模块化设计**：`fetch -> clean -> dedupe -> summarize -> publish` 每个阶段独立函数，便于单独测试和替换。
- **保留原文链接**：摘要只是入口，不让 AI 替代阅读原文。
- **监控失败源**：每个 source 记录最近抓取成功时间、条目数、错误信息。连续失败超过阈值时告警。
- **利用 MCP 做按需查询**：不要让 Agent 只能被动接收日报。把摘要库抽象成 MCP 工具，能提升信息复用率。
- **控制推送频率**：信息流管线的目标不是把噪声从 RSS 搬到 IM，而是把噪声降下来。日报或任务触发式查询通常足够。

## 总结

RSS + AI 摘要这条管线技术上并不复杂，复杂的是如何让它在真实 feed 的混乱里稳定运行。关键不在模型能力，而在过滤、清洗、幂等和可观测性。把每个环节做扎实，再通过 MCP 暴露给 Agent，就能得到一条低成本、可自控、可复用的信息流基础设施。

---

