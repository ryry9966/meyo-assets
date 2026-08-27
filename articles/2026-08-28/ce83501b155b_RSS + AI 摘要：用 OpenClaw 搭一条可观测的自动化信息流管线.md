---
title: RSS + AI 摘要：用 OpenClaw 搭一条可观测的自动化信息流管线
feedId: 34996
source: 综合讨论
publishedAt: 2026-08-28
---

信息过载不是新问题，RSS 依然是低噪声信息入口。但“订阅 50 个源、每天看几百条”不可持续。AI 摘要能降低阅读成本，但直接抓 RSS 丢给模型，往往得到的是格式混乱、重复、成本失控的输出。本文记录我在 OpenClaw 下搭 RSS + AI 摘要管线的做法，重点在工程化和排障，而不是提示词技巧。

## 背景与问题

RSS 源质量参差：RSS 2.0/Atom 混杂，部分源没有 `content:encoded`，只有摘要；部分源时间戳缺失；链接是相对路径。直接逐条喂给模型还有几个明显问题：

- 重复内容：同一事件多源报道，或同一源重复抓取。
- 长文摘要不稳定：直接截断会丢尾部关键信息。
- 推送噪音：单条推送会造成打扰，汇总度低。
- 失败不可见：抓取失败、提取失败、摘要失败混在一起，难以定位。

因此需要一条可拆分的管线：采集 → 标准化 → 去重 → 正文提取 → AI 摘要 → 推送/入库。

## 做法

我把每个阶段做成独立工具或脚本，OpenClaw 负责调度和串联。也适合以后封装成 MCP 工具使用。

### 1. 抓取与标准化

用 `feedparser` 或 `requests + lxml` 读取源。统一字段：`title`、`link`、`published_at`、`summary`、`content`。关键操作：

- 用 `urljoin` 处理相对链接。
- `published_at` 缺失时使用抓取时间，并标记 `is_fallback_time=true`。
- 原始内容 archive 一份，便于回滚和重跑。

### 2. 去重

去重分两层：先做 URL 规范化，去掉 `utm_*` 等跟踪参数；再计算 content 或 title 的短 hash 存入 SQLite。跨源重复用标题相似度处理，阈值我用 `difflib` 0.9，只合并高相似标题。这样能挡住大部分重复，又不至于误杀同主题不同角度的文章。

### 3. 正文提取

优先取 `content:encoded`；为空则抓原文，使用 `readability` 提取正文，去掉 `script/style/nav`。长度限制在 4000–6000 字符，保留开头和结尾，中间可跳读。若提取失败，降级使用源的 `summary` 字段，并标记 `extract_status=fallback`。

### 4. AI 摘要

批量调用模型，不要把每条都单独推。提示词固定为：基于原文提取 3–5 条事实性要点，每条不超过 40 字；不推断、不补充；原文未提及就写“未提及”。如果运行在 Agent 下，可以在摘要前加一步 relevance 判断：根据关注列表判断是否值得进入简报，减少无关内容。

### 5. 推送

聚合为一条 digest，按源分组，标题加日期和来源。支持 dry-run，输出到文件或日志而不推送。正式推送失败重试一次，避免重复轰炸。

伪代码大概是这样：

```python
for feed in feeds:
    items = fetch(feed)
    items = normalize(items)
    items = dedupe(items, db)
    for it in items:
        content = extract_content(it)
        summary = llm_summarize(content)
        db.save(it, content, summary, status="summarized")

digest = build_digest(db.today())
deliver(digest)
```

## 踩坑点

- **编码与相对链接**：`feedparser` 能处理大部分，但某些源 encoding 声明错误，需要 `requests` 响应后手动 decode。链接统一 `urljoin`。
- **正文提取失败很常见**：不要只依赖 `content:encoded`，很多内容为空或只给前两段。fallback 必须有。
- **token 成本**：长文截断要保留首尾，或做分段摘要再合并。不要无脑全文丢进去。
- **去重阈值**：标题相似度太严格会漏去重，太松会误合并。建议同一源用 content hash，跨源才用标题相似度。
- **模型幻觉**：必须写“不推断”，并要求保留原文链接。摘要结果旁边放 link，便于回溯。
- **推送刷屏**：单条推送会很快被关掉。用每日或半日 digest，并限制每个源最多 N 条。
- **失败静默**：给每条记录维护状态机：`fetched → extracted → deduped → summarized → delivered`。失败原因入库，比如 `fetch_error`、`extract_error`、`summarize_error`。失败率超过阈值再告警。

## 可复用建议

- **模块化**：把 `fetch/normalize/dedupe/summarize/deliver` 拆成独立工具，单独测试。不要写一个难以调试的大脚本。
- **配置化**：`feeds.yaml` 声明源地址、标签、优先级、过滤关键词和摘要长度。
- **保留 raw 与摘要版本**：至少保留原文链接和摘要文本，避免重跑时重复调用模型。
- **先小范围跑**：3–5 个源 dry-run 几天，确认摘要质量和去重效果后再扩大。
- **不要用最强模型做摘要**：普通模型足够，把成本留给真正需要推理的判断环节。
- **把 pipeline 做成 MCP/插件后**，OpenClaw 可以把它当作一个可观察、可干预的工具链，而不是黑盒。

## 总结

RSS + AI 摘要的关键不在“AI 摘要”这一步，而在管道是否健壮。把采集、清洗、去重、摘要、推送拆开，保留状态与失败记录，才能稳定输出可用的每日简报。OpenClaw 适合做这类自动化编排：它把工具调用、调度和记录串起来，让信息流治理从“偶尔跑一次脚本”变成可维护的工程实践。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/fee1c234e967c8f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1a9aada36033a58d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1fd2adc0a2d6cd56.png)

