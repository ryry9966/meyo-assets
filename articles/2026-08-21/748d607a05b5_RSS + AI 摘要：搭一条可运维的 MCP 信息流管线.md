---
title: RSS + AI 摘要：搭一条可运维的 MCP 信息流管线
feedId: 33967
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

RSS 没死，但原始订阅已经不适合直接进入 Agent 上下文。一个源一天几十条，十条源就是几百条；全文质量参差、重复转载多、混着活动帖和招聘帖。AI 摘要能降噪，但如果让 OpenClaw 或通用 Agent 直接读 feed，通常会遇到三个问题：上下文爆炸、重复消费、不可观测。

这篇不聊“AI 自动读日报”的魔法，只讲怎么用工程化方式把 RSS + AI 摘要做成一条可复用的管线。

## 问题拆解

信息流管线要解决的不是“能不能摘要”，而是：

- 同一篇文章在不同源里反复出现；
- RSS 的 `guid` 不稳定，不能只靠它去重；
- 全文可能 5 千字，也可能只有一段描述；
- 摘要结果要能被 Agent 稳定消费，不能每次都是自由格式；
- 失败、超时、重复推送要有记录。

所以核心不是 Prompt，而是状态管理、边界控制和固定数据结构。

## 做法

我把管线拆成四层：采集、抽取、摘要、投递。OpenClaw 只负责编排和规则触发，不直接接触原始 HTML。

### 1. 采集层

用 `feedparser` 或 RSSHub 统一输出结构化 JSON。每条只保留：

```json
{
  "feed": "source-name",
  "link": "https://example.com/post/123",
  "title": "Post title",
  "guid": "maybe-unstable",
  "published": "2025-04-01T08:30:00Z",
  "description": "..."
}
```

设置 UA、超时 15 秒、失败重试一次。采集层不做任何智能判断。

### 2. 抽取与去重层

去重键用 `hash(link + title)`，不要只信 `guid`。存储到 SQLite 或 Redis，推荐 SQLite 单文件，方便排查。

需要全文时用 `trafilatura` 或 `readability` 抽取正文；失败降级到 RSS `description`。全文抽出来后截断到 4000～6000 tokens，避免摘要成本失控。

### 3. 摘要层

先粗筛，再精读。粗筛可以用小模型或规则：标题命中关键词、来源权重、发布时间窗口。精读让 LLM 输出固定 JSON：

```json
{
  "title": "...",
  "one_line": "...",
  "key_points": ["...", "..."],
  "action_needed": false,
  "tags": ["ai", "mcp"],
  "score": 0.7
}
```

Prompt 里明确：只输出 JSON，不要解释，不要补链接以外的信息。这样 Agent 拿到的不是散文，而是可过滤、可排序的结构化对象。

### 4. 投递层

生成每日/每半日 digest，推送到 Telegram、ntfy 或邮件。投递前检查 `sent_at`，已经发过的指纹不再重复推送。

## MCP 接入

我把三个动作暴露成 MCP tools：

- `fetch_rss(url) -> items`
- `summarize_item(item) -> summary_json`
- `push_digest(items) -> status`

OpenClaw 里只需要配置触发周期和规则，例如：只推送 `score >= 0.6` 且 `action_needed == true` 的条目；低分但高频出现的主题，进入另一条慢速通道。

这样 Agent 的上下文只会收到最终 JSON，不会吞原始 HTML。

## 踩坑点

- **guid 不稳定**：有些源每次抓取都变。用 `link + title` 哈希做去重更可靠。
- **全文过长**：直接全文摘要容易超时或消费大量 token。先截断，再决定是否需要二次抽取。
- **重复推送**：去重放在了抽取层，但投递层也要有幂等。否则 LLM 重试会造成同一内容发两次。
- **时间混乱**：一律转 UTC 再比较。RSS 里的时区经常不可靠。
- **Prompt 输出漂移**：不加 JSON Schema 时，模型偶尔会加 Markdown 代码块或自由发挥。让 MCP tool 做一次 JSON 校验，失败就重试或丢弃。
- **空摘要**：某些技术博客 description 只有一句话，全文又抓不到。这时标记为 `low_quality`，不进入 AI 精读，避免浪费。

## 可复用建议

- 状态表尽量简单：`fingerprint TEXT PRIMARY KEY, status TEXT, summarized_at TEXT, sent_at TEXT`。
- 摘要结果保留最近 7 天，方便回溯 Agent 的决策。
- 模型分层：本地小模型做过滤和打分，云端模型只处理通过初筛的条目。成本可以降一半以上。
- 监控三个指标：空摘要率、重复率、LLM 平均耗时。不要只监控“有没有推送成功”。
- Prompt 模板和 JSON Schema 版本化。变更摘要结构时，老摘要要有兼容字段。

## 总结

RSS + AI 摘要不是“喂给模型就能自动读”，而是一条需要边界和状态的管线。采集、去重、截断、结构化输出、幂等投递，每一步都在控制 Agent 的上下文和成本。OpenClaw 在这个链路里适合做编排层，把 MCP tools 串起来，而不是把所有内容塞进一个 Prompt。

把读和摘要分开，把状态管起来，这条信息流才能真正稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/8ff4f0a3b0135545.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/2dd43249bd16d6c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/c6648fc0cae1eff7.png)

