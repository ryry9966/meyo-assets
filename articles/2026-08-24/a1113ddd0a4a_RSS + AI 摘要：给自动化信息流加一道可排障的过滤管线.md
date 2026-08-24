---
title: RSS + AI 摘要：给自动化信息流加一道可排障的过滤管线
feedId: 34565
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

RSS 在 Agent/MCP 工具链里仍然很实用。很多团队用 RSSHub 把公众号、论坛、发布页转成 RSS，再交给自动化流程消费。但直接把几十个源喂给 Agent 或人工阅读，问题很明显：全文长、噪音多、Token 成本高，而且不少 RSS 源只给标题和摘要，点进去才是正文。

我实践的方案不复杂：RSS 抓取 -> 清洗去重 -> AI 摘要 -> 结构化输出 -> 下游消费。这篇文章把这套管线的关键决策和坑写下来，适合已经接触 OpenClaw/Agent 编排或 MCP 的实践者。

## 问题定义

先明确目标产物：不是全量正文入库，而是得到一种“可检索、可溯源、低噪音”的条目。建议统一成这个 schema：

- title
- link
- published_at
- summary_points: 3-5 条要点
- tags
- generated_at
- content_hash

没有固定 schema，AI 摘要很容易变成不可控段落，下游 Agent 更是难以消费。结构化输出建议直接落到 JSON 或 Markdown，不要输出自然语言报告。

## 做法/步骤

**1. 选抓取层**

不建议自己写 HTTP 抓 XML。推荐 Miniflux 或 FreshRSS 做订阅管理，RSSHub 负责把非 RSS 源转成 RSS。它们自带重试、feed 级错误隔离和 API，能省大量编码、相对链接、RSS 2.0/Atom 兼容工作。

**2. 定时任务**

在 OpenClaw 或 cron 里每 30 分钟跑一次。不要每分钟跑，很多源不会那么快更新，频繁抓取容易被限流。任务入口加锁，避免上一轮没跑完又触发新任务。

**3. 清洗与去重**

入库前做标准化：统一时间字段、补全相对链接、去除 HTML 标签、合并重复条目。去重不建议只靠标题，标题经常重复。可以用 `link` 或 `title + published_at + source` 生成 sha1 指纹，再入库判断。

**4. AI 摘要**

把清洗后的条目发到 OpenAI-compatible API 或本地 Ollama。Prompt 要点：

- 只依据原文摘要/标题，不要补充外部知识；
- 输出 JSON 对象，字段固定；
- 如果信息不足，summary 写“信息不足”，不要编造；
- 保留原文链接。

温度设 0 或 0.1。输出不稳定时，在 prompt 里给一个完整 JSON 示例，并在代码层做 schema 校验，解析失败重试一次，再失败进死信队列。

**5. 下游消费**

产出 JSON 后可以写 Markdown 到 Obsidian，也发到消息通道，或让 OpenClaw 的 MCP 工具读取。如果给 Agent 当上下文，建议只发摘要和原文链接，不要附全文，避免 token 浪费。

## 踩坑点

- **RSS 源格式脏**：有些源 title 重复、缺少 pubDate、内容是一张图。解析层要容错，缺时间的用抓取时间兜底。
- **相对链接和 HTML entity**：必须用解析器还原绝对 URL，否则去重和回链会失效。
- **AI 幻觉**：模型容易“补充背景”。Prompt 要强调只做摘录；我额外要求 summary 里不得出现原文没有的具体数字。
- **输出字段漂移**：模型可能换成大写字段，或把 JSON 包在 ``` 里。解析时先剥离代码块，再用宽容 JSON 解析；schema 不过就重试或降级为空摘要。
- **任务整体失败**：一个 feed 坏掉不应阻塞全部。fetch/parse 阶段按 feed 独立 try/catch，失败写日志并继续。
- **时区问题**：pubDate 有 UTC 也有本地时间。统一转 UTC ISO 8601，否则排序和去重都会乱。

## 可复用建议

- 把管线拆成 fetch / normalize / summarize / sink 四层，层与层只用统一 schema 连接。换源、换模型都不影响其它层。
- 配置外置，源列表、模型名、API base、输出路径、通知 webhook 都从配置读，不要硬编码。
- 先跑 dry-run，只打印不写入，至少调三天 prompt 再开自动推送。
- 保留原始条目和生成结果，方便回溯。一个 content_hash 对应 raw 和 summary 两份。
- 在 OpenClaw 里可以把这套管线包装成定时 skill，摘要结果通过 MCP 暴露给其它 Agent，源头始终保留原文链接作为 ground truth。

## 总结

RSS + AI 摘要的价值不在“自动看新闻”，而在给 Agent 提供低噪音、可溯源、可约束格式的信息输入。把清洗、去重、schema 校验做扎实，比换更大模型更有效。先小范围跑通，再扩大源列表，会比一次性接入几十个源更可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/26359cbe0a0866c8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/23c2e349bd81bcc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/418792383a6e30ab.png)

