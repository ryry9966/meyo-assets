---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 34139
source: 综合讨论
publishedAt: 2026-08-22
---

我已经不再追求“更多信息”，而是想少看一点。RSS 是少数能自己掌控的信源，但订阅数一多，未读列表很快变成另一种焦虑。于是把 RSS 和 LLM 摘要接成一条自动管线：每天只读一份 digest，代替逐条翻原文。

这篇文章记录我在 OpenClaw/Agent 环境里落地这条管线的过程，不涉及复杂框架，核心是抓取、清洗、去重、摘要、推送这几个环节。

## 背景

信息过载不是新鲜问题。RSS 本身已经做了一层信源过滤，但原始条目仍然太多：标题党、重复报道、低相关度内容混在一起。直接让 LLM 处理一堆未经清洗的 RSS 内容，效果往往很差，上下文容易爆，输出也不稳定。

所以目标很明确：把 RSS 变成可靠的、可消费的摘要流，而不是另一个需要人工维护的未读列表。

## 问题

直接把 RSS 全文丢给模型会踩到几个典型坑：

- RSS 源只给摘要，正文缺失或截断；
- HTML 夹杂脚本、广告、Cookie 弹窗；
- 同一篇文章在不同链接参数下重复出现；
- 时间字段时区不一致，导致增量判断错误；
- LLM 输出格式不稳定，JSON 解析失败；
- 推送接口限流或超时，导致重复推送。

这些问题不是模型能力问题，而是数据管道问题。下面按步骤拆解。

## 做法 / 步骤

### 1. 抓取与配置

使用 `feedparser` 或 RSSHub 获取源。源列表放到 YAML 配置里，不要写死在代码中：

```yaml
sources:
  - name: hackernews
    url: https://hnrss.org/frontpage
    max_items: 20
  - name: tech_blog
    url: https://example.com/feed.xml
    max_items: 15
```

每个源独立配置，便于单独调试。

### 2. 清洗与归一化

对每个 entry 提取统一字段：`title`、`link`、`published`、`summary/content`。用 `BeautifulSoup` 或 `lxml` 把 HTML 转纯文本，移除 `script`、`style`、`nav`、`footer`。正文优先取 `content`，没有则用 `summary` 做 fallback，并记录字段来源。

### 3. 去重与增量

用 SQLite 存处理历史。关键是 hash 策略：不要只用 URL，因为 `utm_source` 这类参数会让同一篇文章出现多个 URL。建议对“规范化 URL + 标题”或正文前 2KB 做 `sha1`，存为 `content_hash`。每次抓取后先查重，已存在则跳过。

SQLite 表结构可以很简单：

```sql
CREATE TABLE seen_items (
  hash TEXT PRIMARY KEY,
  source TEXT,
  title TEXT,
  link TEXT,
  first_seen_at TEXT
);
```

### 4. AI 摘要

先过滤相关性，再摘要，比一步到位更稳。Prompt 示例：

```text
你是信息过滤助手。请先判断以下文章与“AI 工程化、自动化、开源工具”是否相关。
相关则输出不超过 80 字的中文摘要，并给出 1-3 个关键词；无关输出 SKIP。
标题：{title}
正文：{content}
```

控制每条正文长度，超过模型上下文就截断。批量处理时加并发限制和重试退避。

### 5. 输出与推送

生成 Markdown digest，按源分组，每小时或每天推送一次。推送渠道可以是 Webhook、飞书、邮件，也可以写文件供 Agent 读取。

### 6. 调度

用 cron 或 systemd timer 定时执行。如果已经在 OpenClaw 或类似 Agent 环境里跑自动化，可以把摘要工具包装成 MCP server 暴露出去，让 Agent 按需拉取和总结，而不是只能全自动推送。

## 踩坑点

- **RSS 源质量参差**：很多站点只给摘要，不给全文。需要为不同源配置正文获取策略，比如抓原文或只摘要已有内容。
- **HTML 清洗不彻底**：脚本、cookie 弹窗、导航栏会污染正文。清洗规则要按源调整，必要时保留原始 HTML 做回溯。
- **时间解析混乱**：用 `dateutil.parser` 并强制转 UTC，否则增量判断会漏或重复。
- **重复条目**：URL 参数变化、同一内容转发到多个源，都会导致重复。hash 策略比 URL 去重更可靠。
- **LLM 输出不稳定**：要求 JSON 时用 `json_repair` 兜底，或让模型直接输出 Markdown 再解析，减少解析失败。
- **限流与成本**：一次拉几百条摘要，注意模型 rate limit 和 token 消耗。用并发控制、缓存摘要结果、必要时选择本地小模型降低成本。
- **推送不可靠**：推送接口超时或限流时，要记录失败状态，避免下次重复推送同一条。

## 可复用建议

- **模块化**：fetcher、cleaner、deduper、summarizer、notifier 各自独立，替换某一块不影响整体。
- **配置化**：源列表、过滤关键词、推送渠道都放配置，改配置不用改代码。
- **原始数据落盘**：保存抓取到的 raw item，便于回溯、重放和调试。
- **dry-run 模式**：跑通预览但不推送，先看摘要质量再开放。
- **日志与指标**：记录每源抓取数、去重后数、摘要成功/失败数，方便排查。
- **Prompt 版本化**：摘要 prompt 要版本管理，避免改 prompt 后历史数据不可比较。
- **失败不阻塞整体**：单个源失败不影响其他源，单个条目摘要失败可以标记后跳过。

## 总结

RSS + AI 摘要管线技术难度不高，难点在边界情况。先跑通最小闭环，再根据真实源不断加固。稳定性优先于智能，数据质量和状态管理才是长期可维护的关键。对 AGENT/自动化用户来说，把这条管线跑稳之后，它就是一个可靠的“信息预处理层”，上游可以接任意推送或 Agent 决策。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/89d77be93c991abd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/bb2b176140ca350b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/89251656ef9dea59.png)

