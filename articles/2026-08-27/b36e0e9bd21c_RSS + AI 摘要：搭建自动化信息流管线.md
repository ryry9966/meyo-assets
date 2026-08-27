---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 34939
source: 综合讨论
publishedAt: 2026-08-27
---

## 背景

RSS 依然是从技术博客、产品更新、安全公告获取一手信息的高性价比渠道。订阅源涨到几十个后，未读列表会变成一种背景焦虑：每天点开 50 篇，多数不想深入读，但偶尔又怕漏掉关键内容。用 AI 做摘要看起来合理，但直接“每条都丢给大模型”很快会遇到 token 消耗、重复条目、输出噪音和推送疲劳。

## 问题

要工程化地解决的不是“能不能摘要”，而是：

- 如何低成本过滤不值得读的条目；
- 如何避免同一条目因 feed 更新、URL 参数变化被重复摘要；
- 如何让摘要格式稳定、适合在消息工具里快速扫读；
- 管线挂了怎么被发现、怎么重跑。

## 做法/步骤

### 1. 组件选型

- RSS 解析/订阅：Miniflux、FreshRSS，或者直接解析 XML。Miniflux 的 API 适合自动化，自带抓取历史和状态。
- AI 摘要：任意 OpenAI-compatible API，或本地 Ollama、vLLM。用统一接口，避免绑定某个供应商。
- 推送：Telegram、飞书、企业微信、Slack Webhook 都可以，建议走 webhook。
- 调度：cron 或自托管 worker；开发阶段先用手动 dry-run。

### 2. 管线设计

抓取 -> 标准化 -> 规则过滤 -> 去重 -> AI 摘要 -> 格式化 -> 推送。

标准化是容易忽略的一步：来源可能给相对时间、不同时区、HTML 实体、奇怪编码。建议先统一成 UTC 时间戳、去 HTML 标签、限制标题长度。

### 3. 一个最小实现

```python
from datetime import datetime, timedelta
from hashlib import sha1

entries = miniflux.fetch_entries(since=datetime.utcnow() - timedelta(hours=3))
entries = [e for e in entries if passes_rules(e)]  # 来源白名单、关键词、长度
entries = dedupe(entries, key=lambda e: sha1(e["url"].split("?")[0].encode()).hexdigest())

for e in entries:
    summary = ai.chat(
        prompt=(
            "Summarize the following article in 3 bullets. "
            "Keep each bullet under 25 words. Do not add facts not present. "
            "If the article is a changelog, prioritize breaking changes and actions.\n\n"
            f"Title: {e['title']}\nContent:\n{e['content'][:2000]}"
        )
    )
    send_webhook(format_message(e, summary))
    mark_processed(e["id"])
```

### 4. 摘要 prompt 设计

不建议让模型自由发挥。固定“标题 + 三点要点 + 行动项”的模板，限制每条不超过 25 词，并明确要求不要补充原文没有的事实。内容截断到 1500–2500 字符即可，大部分技术文章的信息密度足够。

### 5. 状态管理

用 SQLite 或 KV 保存已处理条目的 `hash(url)`、处理时间、推送状态。URL 去重时建议去掉 `?utm_source=` 这类跟踪参数。对于同一篇文章的更新，可以按 `guid` 去重，而不是 URL。

## 踩坑点

- **全量摘要太贵**：订阅源里可能有大量灌水、转载、社交媒体。先做规则过滤，比如只保留技术域、标题含关键词、正文字数超过 300 字。粗筛后能削减 60–80% 的无效请求。
- **模型输出不稳定**：温度设高后可能给出 markdown 列表、加戏、翻译腔。建议 temperature 0.1–0.2，要求返回纯文本或 JSON。若用 JSON，要处理解析失败，失败时回退到原文链接。
- **推送被限流**：一次跑 50 条时不要逐条发送，合并成一条摘要卡片，或每 5 分钟发送一批，避免被 webhook 平台临时封禁。
- **时区和数据窗口**：不要用本地时间算 `since`。用 UTC 或直接用 Miniflux 的 entry id 游标，避免夏令时切换重复抓取。
- **模型幻觉**：摘要再漂亮也必须附带原文链接。否则出现技术参数被改动时很难回溯。

## 可复用建议

- 把这条管线封装成 MCP 工具或 plugin，例如 `rss_fetch`、`rss_summarize`、`rss_push`，方便 OpenClaw/Agent 调用。这样后续可以用自然语言触发“帮我总结最近 3 小时的重要 RSS”。
- 配置与代码分离：来源列表、规则、提示词、推送目标放在 YAML/JSON，不要硬编码。
- 先跑 dry-run 一周，看看过滤规则是否合理，再开启自动推送。
- 监控失败：管线本身要有一个“自检” webhook，失败时推送到另一个渠道，而不是只在标准输出里打日志。

## 总结

RSS + AI 摘要的难点不在模型，而在信息流的“管道”：去重、过滤、重试、格式稳定。把每个环节做成独立、可替换的组件，比一次写完的脚本更有长期价值。对 OpenClaw 用户来说，这条管线也可以作为 Agent 的信息入口，后续接入自动打标、归档、触发工单都顺理成章。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/d3844eb3d31f6b6c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/65790d8e93bda04d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/47a377111b6c13dc.png)

