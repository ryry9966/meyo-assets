---
title: RSS + AI 摘要：搭建一条不靠通用 Agent 硬读的信息流管线
feedId: 35476
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

RSS 没有死，反而成了很多技术人员获取信息的最小公约数。订阅几十个源之后，未读数很快会堆积起来，真正值得读的往往只有一小部分。AI 摘要可以把“扫标题”和“读正文”之间的时间压缩掉，但要稳定落地，不能只是让一个通用 Agent 去硬读一串 RSS 链接。

常见的失败形态是：Agent 一次读太多条目导致上下文爆炸；同一个条目反复抓取；正文里混入导航栏和评论区；摘要格式每次都不一样；遇到抓取失败时，模型还可能一本正经地编内容。

所以问题不是“能不能让 AI 读 RSS”，而是怎么把这条链路拆成可监控、可复用、可回滚的工程管线。

## 问题拆解

一条可靠的 RSS + AI 摘要管线需要解决五件事：

1. 稳定抓取与解析 RSS/Atom
2. 提取正文，而不是只读 feed 里的摘要
3. 去重与状态持久化
4. 输出可解析的结构化摘要
5. 低成本地批量调用与推送

这五件事大部分都不该由 LLM 来做。LLM 只适合干摘要、筛选和打标，其它环节应该交给确定性的工具。

## 做法 / 步骤

### 1. 拆成工具，而不是一个端到端 Agent

把能力拆成独立函数或 MCP 工具，能极大降低调试成本：

- `list_feeds()`：读取订阅源配置
- `fetch_feed(url)`：抓取并解析 RSS/Atom
- `extract_article(html)`：从原文页抽取正文
- `dedupe(entries)`：按 guid/link 哈希去重
- `summarize_batch(items)`：批量生成结构化摘要
- `mark_processed(ids)`：记录已处理条目

Python 里用 `feedparser` 解析订阅源，用 `trafilatura` 抽取正文，足够覆盖大多数场景，不需要重型爬虫。

### 2. 抓取与正文提取

`feedparser` 能处理大部分 RSS/Atom。注意设置 10–15 秒超时，带上合理的 User-Agent。拿到 entry 后，如果 `content` 或 `summary` 字段过短，再请求原文并用 `trafilatura.extract()` 提取正文。

提取前可以做一次预处理：去掉 script、style、导航、社交分享区。提取后的文本落库时保留原始版本，方便以后重新清洗或重新摘要。

### 3. 去重与持久化

用 SQLite 存状态是最省事的选择。表结构大致如下：

```sql
CREATE TABLE entries (
  id INTEGER PRIMARY KEY,
  feed_url TEXT,
  item_id TEXT,
  link_hash TEXT,
  title_hash TEXT,
  raw_content TEXT,
  extracted_text TEXT,
  summary_json TEXT,
  status TEXT,
  created_at TEXT
);
```

`item_id` 存在时直接用；缺失时用 `link` 的 SHA-256；再不行才退化成标题 + 发布时间的哈希。对 link 先做一次规范化，去掉 `utm_*` 之类的跟踪参数，否则容易重复。

### 4. 结构化摘要

不要给模型一个模糊的“总结一下”。明确约束输出字段：

```json
{
  "headline": "一句话结论",
  "key_points": ["要点1", "要点2"],
  "relevance_score": 8,
  "action_items": [],
  "language": "zh"
}
```

批量处理时，5–10 条一组放进一个请求。提示词里要求只输出 JSON，不要额外解释。如果 API 支持 JSON mode 或 function calling，优先打开。解析前先剥掉可能的 markdown 代码块标记，再 `json.loads()`。

### 5. 推送与闭环

摘要写回数据库后，通过 webhook 推到 Slack、ntfy 或邮件。推送格式建议是：“标题 + 一句结论 + 2–3 个要点 + 原文链接”。不要贴全文，否则摘要就失去意义了。

### 6. 调度

定时任务用 systemd timer 或 cron，每小时跑一次。失败重试用简单指数退避，单条失败不影响整批。日志里记录 feed_url、失败原因和重试次数，方便后面排障。

## 踩坑点

- **正文提取脏**：很多技术博客页面包含代码块、评论区、相关推荐。`trafilatura` 默认参数对多数页面够用，但某些站点需要自定义 XPath。
- **编码与超时**：部分中文老博客不是 UTF-8，需要检测编码。请求超时调短一点，避免整个任务被某一个慢源卡死。
- **重复条目**：有些 feed 的 `guid` 不稳定，或 link 带不同查询参数。对 link 做规范化哈希比单独依赖 `guid` 更稳。
- **AI 输出漂移**：模型可能在 JSON 外包一层 markdown 代码块，或者输出注释。解析前做一层清洗。
- **成本与上下文**：长文直接塞进 prompt 很耗 token。先截断正文（保留前 2000–3000 字），或分段摘要再合并。对低相关度条目不要做全文摘要。
- **不要让 Agent 全自动**：多步骤任务一旦出错，Agent 可能幻觉出内容。把抓取和摘要分开，工具返回真实数据，LLM 只做摘要和筛选。

## 可复用建议

如果要在 OpenClaw 或 MCP 生态里复用，建议直接做成 MCP server，把上面几个函数暴露成工具。交互式排查时可以直接调用单个工具，而不是每次跑完整流程。

另外建议做分级处理：先跑标题级过滤，让 LLM 给标题打分，只对高分条目提取正文和全文摘要。这样通常能省下 70% 以上的 API 调用。

不同订阅源可以配置不同摘要模板。技术博客重点提取代码改动、版本号和 breaking change；新闻源重点提取事件、影响和时间线。

## 总结

RSS + AI 摘要的稳定方案不是一个黑盒 Agent，而是一条可拆解、可监控的管线：抓取 → 清洗 → 去重 → 摘要 → 推送。把每步做成独立工具，LLM 只在摘要和筛选环节介入，状态落到 SQLite，不但稳定，而且容易回溯和调试。下一步可以考虑加个人兴趣相关度打分，或者用本地小模型降低长期运行成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/5ef8343d64ffa516.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/59e35a04e2217f0e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/40a464763ad26324.png)

