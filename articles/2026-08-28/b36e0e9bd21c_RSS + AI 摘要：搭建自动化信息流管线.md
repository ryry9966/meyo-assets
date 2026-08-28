---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 35104
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

我自己的信息源里，RSS 仍然是最稳定的一层：博客、技术周刊、安全公告、产品更新都能落到同一个入口。但问题也很明确：源一旦超过 20 个，每天新增条目很容易上百，未读数字从“待办”变成了“背景噪声”。后来我把 AI 摘要接到 RSS 后面，目标不是让模型替我看完所有内容，而是把每条压缩成“一句话 + 关键点 + 是否值得读”，只推送高相关条目。

## 问题

直接让 Agent 每篇都抓全文再总结会踩几个坑：

- 全文喂给 LLM 成本高、延迟大，很多 RSS 正文夹杂导航栏、推荐位和脚本；
- 源格式混乱，guid 不稳定，重复条目多；
- 推送失败时如果状态设计不好，会反复推同一批；
- 源失效或抓取超时会影响整条管线。

所以这套东西不能只靠一个 prompt，需要拆成有状态的管线。

## 做法 / 步骤

整体结构：

RSS 源 -> 抓取器 -> 标准化 -> 去重/过滤 -> AI 摘要 -> 推送

在 OpenClaw 里我把它拆成几个 MCP 工具：`rss_fetch`、`rss_normalize`、`rss_dedup`、`ai_summarize`、`notify`。Agent 只负责调度和异常处理，不保存业务状态。

### 1. 源配置

用 YAML 维护源列表，每个源可独立配置：

```yaml
feeds:
  - name: solidot
    url: https://www.solidot.org/index.rss
    max_items: 10
    timeout: 10
    enabled: true
  - name: hn-frontpage
    url: https://hnrss.org/frontpage
    max_items: 15
    timeout: 10
    enabled: true
```

### 2. 抓取与标准化

每个源单独抓取，设置 UA 和超时。XML 解析时强制 UTF-8，忽略非法字符。解析后统一成内部结构：

```json
{
  "id": "hash(link)",
  "title": "文章标题",
  "link": "原文链接",
  "summary": "feed 自带摘要",
  "content": "提取后的正文，截断到 2500 字符",
  "published_at": "2025-01-01T00:00:00Z",
  "source": "solidot"
}
```

正文优先用原文提取，失败则降级到 feed 自带 summary。不要追求正文完整，摘要场景下 2000-3000 字符足够。

### 3. 去重与过滤

去重不要依赖 RSS guid，很多源会变。用 link 做规范化后 hash，存 SQLite：

```sql
CREATE TABLE processed_items (
  link_hash TEXT PRIMARY KEY,
  source TEXT,
  created_at TEXT
);
```

进入 LLM 之前先做粗筛：域名黑名单、关键词黑名单、重复标题。这样可以省掉相当一部分 token。

### 4. AI 摘要

批量处理，每批 5 篇，结构化输出：

```json
{
  "title": "原标题",
  "one_sentence": "一句话摘要",
  "key_points": ["要点1", "要点2", "要点3"],
  "relevance_score": 8,
  "should_read": true
}
```

Prompt 里明确“只输出 JSON，不要输出 Markdown”，温度设低，开启 JSON mode。相关性分数可以根据你的兴趣领域动态生成，比如安全、AI infra、开源工具。

### 5. 推送

只推 `should_read = true` 或 `relevance_score >= 7` 的条目。推送模板示例：

```markdown
**Solidot**
- [标题](链接)
  一句话摘要：...
  要点：...
  相关度：8/10
```

通过 webhook 推企业微信/飞书/Discord，或者邮件。

OpenClaw 任务流可以这样配置：

```yaml
tasks:
  - name: rss-digest
    trigger: cron(30 * * * *)
    steps:
      - use: mcp.rss_fetch
      - use: mcp.rss_normalize
      - use: mcp.rss_dedup
      - use: mcp.ai_summarize
      - use: mcp.notify
```

## 踩坑点

- XML 编码声明和实际内容不一致：解析时用 `errors="replace"`，不要信任头部声明。
- guid 不稳定：不要拿 guid 当主键，用 `canonicalize(link)` hash。
- 时区混乱：RSS 里的 `pubDate` 可能是 GMT、CET 或没时区，统一转 UTC ISO 再入库。
- 正文提取失败：不要中断整个源，降级为 feed 自带 summary；如果 summary 也空，只推标题。
- LLM 输出超长：限制输入 2500 字符，`max_tokens` 控制到 300 左右，否则 20 个源一天 token 消耗会失控。
- 推送失败导致重复：状态机用 `pending -> sent`，推送成功才写 `sent`；失败保留 `pending`，下次重试。
- 源失效：定期检查每个源的最后成功时间和新增数量，连续 3 次失败自动禁用并通知。

## 可复用建议

- 不要做“AI 替你看”，做“AI 帮你筛选”。保留原文链接，让最终判断回到人。
- 摘要输出固定 JSON，比散文更容易解析和推送。
- 把抓取、去重、推送做成 MCP 工具，而不是写在 Agent 的 system prompt 里。工具边界清晰，更新替换成本低。
- 状态放外部存储，不要放 agent memory。SQLite 足够。
- 每个源独立适配器，可单独超时、限流、禁用。
- 记录每天新增、过滤、推送、失败数量，以及 token 消耗。没有监控的管线最后都会变成黑盒。

## 总结

RSS + AI 摘要的收益不在模型多聪明，而在于管线稳不稳定、成本是否可控、推送是否克制。把这套拆成标准化、去重、摘要、推送四个环节，逐步叠加规则，比一次性让 Agent 读全文要可靠得多。稳定运行之后，它才会从“又一个自动化脚本”变成真正能降低信息负担的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1d79b86e04f33142.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/253c27535fe53675.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/a4ef888bf9cdb8c3.png)

