---
title: RSS + AI 摘要：搭建一条可维护的自动化信息流管线
feedId: 35183
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

RSS 对自动化用户来说仍然是最可控的信源之一：不需要逆向接口、不受推荐算法干扰、源列表自己维护。但订阅数量一多，问题也很直接：原始条目太长，直接喂给 Agent 会爆上下文；很多源重复、标题党、编码混乱，摘要质量不稳定。我们需要一条干净的管线：抓取、清洗、去重、AI 摘要、结构化输出、归档或通知。

这套做法不依赖特定大模型，也不需要复杂平台，配合 OpenClaw/Agent/MCP/插件都能接上。

## 问题

先明确要解决的四件事：

1. **信息过载**：每小时可能新增几十条，不能全部进上下文。
2. **重复与噪音**：同一内容在不同源反复出现，或者标题党文章混入。
3. **上下文成本**：原文过长，直接摘要浪费 token，也容易让 Agent 抓不住重点。
4. **可回滚与可维护**：摘要格式、源列表、调度规则需要可追溯。

## 做法/步骤

### 1. 固定信源，标记频率

建议从 10–30 个高质量源开始，不要无脑订阅。用配置文件维护：

```yaml
sources:
  - name: ai-eng
    url: https://example.com/feed.xml
    category: engineering
    max_items: 10
  - name: security
    url: https://example.org/rss
    category: security
    max_items: 15
```

### 2. 抓取与清洗

在 OpenClaw 的定时任务里触发一个脚本或 MCP 工具。RSS 解析用 `feedparser` 就够了，标题、链接、发布时间、正文或摘要都拿到。HTML 正文先用 `html2text` 或 `BeautifulSoup` 转纯文本，去掉脚本、样式、广告位。

```python
# 伪代码
for entry in feed.entries:
    text = clean_html(entry.content or entry.summary)
    text = normalize_encoding(text)
    items.append({
        "title": entry.title,
        "link": normalize_link(entry.link),
        "published": parse_date(entry.published),
        "text": text[:6000],
    })
```

### 3. 去重与缓存

用 `normalize_link` 作为主键，再加一层内容哈希。很多 RSS 链接带 utm 参数，或者同一篇文章在不同源链接不同，但正文高度相似。简单做法：

```python
key = sha1(normalize_link(link) + title.lower()).hexdigest()
content_hash = sha1(text[:2000].encode()).hexdigest()
```

存储用 SQLite 或 Redis，只处理 24 小时内新增且未见过的新条目。

### 4. AI 摘要

不要给模型全文，除非你确实需要深度阅读。先截断到 6000 字符左右，用结构化 prompt 输出 JSON：

```text
请对以下文章生成摘要，输出 JSON，包含：
- tl_dr: 一句话总结
- key_points: 3-5 个要点
- relevance: 与系统开发/自动化/AI 工程主题的相关性（0-1）
- worth_reading: 是否值得深入阅读
保留原文引用片段，不要编造信息。
```

温度调低，输出用 JSON Schema 校验。失败重试一次，还不行就丢弃并记录。

### 5. 输出与归档

摘要落成 Markdown 或 JSON 文件，按天归档；也可以推送通知，或作为 MCP resource 给 Agent 后续按需读取原文。

## 踩坑点

- **编码混乱**：很多 RSS 声称 UTF-8，实际是 latin-1 或 GBK。抓取时先检测字节，不要盲信 header。
- **重复链接陷阱**：同一篇文章可能链接带 `?from=rss&utm_source=...`，需要规范化 URL 再去重。
- **成本控制**：全量摘要很快烧 token。先按标题关键词、分类、源优先级过滤一批，或限制每批最多摘要 20 条。
- **AI 幻觉**：让模型输出原文引用片段，并强制保留原始链接。否则摘要看起来通顺但不真实。
- **单源故障拖垮调度**：某个源超时、DNS 失败、限流，不能影响整条管线。每个源设置 10 秒超时，失败单独标记，下次重试。
- **Agent 上下文膨胀**：不要直接把 50 条摘要全部塞给 Agent，先让它读索引，再按需取 3–5 条原文。

## 可复用建议

- **保留元数据**：`source`、`fetched_at`、`published_at`、`content_hash`、`summary_version`，方便追查。
- **Prompt 模板版本化**：摘要格式变更时，旧数据仍能对应到具体模板版本。
- **固定输出 schema**：AI 输出必须经过 JSON 校验，脏字段直接丢弃，不要污染后续 Agent。
- **失败隔离与重试**：指数退避，单源失败不影响整体，记录日志即可。
- **先脚本，后 MCP**：如果现有 RSS MCP 不稳定，先用一个 20 行脚本跑通核心流程，再包成 MCP 工具或 OpenClaw 插件，可维护性会好很多。

## 总结

RSS + AI 摘要的价值不在于“自动推送”，而在于控制权：源列表可控、去重可追溯、摘要格式可固定、失败可隔离。先花时间把清洗、去重、schema 校验这些脏活做扎实，再接 OpenClaw/Agent 自动化，管线才会稳定。不要一上来就追求大而全，跑通最小闭环，再逐步加源、加过滤规则、加归档策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/b0fec5c26b0a7f76.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3d96a3bcbff783f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0cbb2e39bc68ca7b.png)

