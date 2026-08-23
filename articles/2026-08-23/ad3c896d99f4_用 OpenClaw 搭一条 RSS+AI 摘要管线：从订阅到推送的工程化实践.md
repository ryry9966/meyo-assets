---
title: 用 OpenClaw 搭一条 RSS+AI 摘要管线：从订阅到推送的工程化实践
feedId: 34362
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

我订阅的 RSS 源有 40 多个，包括技术博客、安全公告、产品更新和行业新闻。每天新增条目 200~400 条，根本看不过来。最早用 RSS 阅读器，后来改成“全量推送”，结果只是把未读压力从阅读器搬到了 IM。

真正需要的是：一条自动化管线，能定时抓取 RSS，过滤掉低价值内容，把长文压成 3~5 行中文摘要，并且只在值得看的时候推送。

## 问题拆解

如果只做“RSS 转 AI 摘要”，很快会遇到几个问题：

1. 很多 RSS 源只提供标题和摘要，没有正文，LLM 只能做标题扩写，价值很低。
2. 全量摘要仍然很多，噪音大，推送频率容易失控。
3. 摘要格式不稳定，有时输出 JSON，有时输出 Markdown，下游解析困难。
4. 重复条目、更新条目、广告和活动帖混在一起，容易误推。

因此管线不能只是“拉取 -> 丢给 LLM”，还需要清洗、过滤、去重、格式约束和失败处理。

## 做法与步骤

### 1. 组件选型

我使用的组合：

- **RSS 拉取**：feedparser（Python）或 rss-mcp server
- **正文提取**：trafileatura / readability-lxml，只在 RSS 缺少正文时触发
- **编排**：OpenClaw 的定时 workflow，接 RSS 工具节点和 LLM 节点
- **摘要**：OpenAI/Claude 兼容接口，使用 JSON mode 输出
- **推送**：Telegram Bot / 飞书 webhook / ntfy，按标签分流

在 OpenClaw 中，管线大致如下：

```
schedule trigger -> rss_fetch(source_list) -> clean/filter -> batch_llm_summary -> rank/filter -> notify
```

### 2. 拉取与清洗

每个源配置成对象：

```yaml
- name: "example_blog"
  url: "https://example.com/feed.xml"
  full_text: false
  tags: ["llm", "agents"]
  max_items: 10
```

拉取后先做基础清洗：

- 去掉 HTML 标签，只保留可读文本
- 标题去除多余空格和乱码
- 生成本地唯一 ID：优先用条目 guid，没有则用 link 做 SHA1
- 增量去重：持久化已处理 ID 到 sqlite 或 KV

如果条目没有正文或正文少于 200 字符，再调正文提取工具抓原文。这里要注意：很多站点有反爬或 Cloudflare，失败就丢弃，不阻塞整个管线。

### 3. 批次摘要与过滤

不要逐条调用 LLM，成本高且慢。我按源或每 20 条一批送入，让 LLM 输出结构化 JSON。

Prompt 要明确：

```
你是信息过滤助手。输入一批 RSS 条目，请完成：
1. 对每条生成 2-3 句中文摘要，保留具体数据、版本号、日期、结论。
2. 判断 relevance_score 0-10，基于用户兴趣：LLM infra、Agent、MCP、自动化。
3. 输出 JSON 数组：[{"id": "...", "summary": "...", "score": 7, "keep": true}]
不要输出 JSON 以外的内容。
```

使用 JSON mode 或 response_format 约束，减少解析失败。如果批量输出仍然截断，可以降低批次大小，或按条重试。

过滤策略：score >= 7 才推送；6~7 进入 digest 汇总，每天推一次；低于 6 丢弃。这样避免 IM 被刷屏。

### 4. 推送格式化

推送内容不要只给摘要，要保留原文链接，并明确标注“AI 摘要”。例如：

```
[AI 摘要] LlamaIndex 发布 0.11 版本
- 新增工作流缓存机制，降低重复执行成本
- 修复了 low-level API 的并发问题
- 相关 MCP 适配仍处于实验阶段
原文: https://...
```

这样即使摘要不准确，用户也可以一键回到原文验证。

## 踩坑点

**RSS 源没有正文**

很多博客 RSS 只输出摘要，甚至只有标题。必须接正文提取工具，但正文提取对 Medium、Substack、JS 渲染站点经常失败。建议 `full_text: true` 的源先尝试 RSS content 字段，失败后再抓原文，并设置 5 秒超时。

**LLM 输出格式漂移**

即使开了 JSON mode，有时返回内容仍会混入 markdown 代码块或额外解释。需要后处理：先剥掉 ```json 围栏，再用 json.loads 解析；解析失败就重试一次，再失败丢弃该批次并记录原始输出，不要让它影响整条管线。

**重复推送**

RSS 源经常把更新条目重新发布，或者 guid 变化。去重不能只靠 guid，要组合 `link + title_hash` 甚至正文前 200 字符 hash。持久化至少 7 天，避免重复推送。

**成本与限流**

如果源很多，全量摘要 token 消耗很大。可以先用标题/关键词规则过滤掉明显无关条目，只对候选条目做 LLM 摘要。比如标题里含“sponsor”“webinar”“survey”的直接降权，不进入 LLM。

**定时任务重叠**

如果上游 RSS 更新慢，一次任务可能跑超过 10 分钟，下次触发又启动。用文件锁或数据库锁，保证同一时间只有一个实例在跑。失败要有退避，不能每分钟重试。

## 可复用建议

- 把源列表、标签、过滤阈值都做成配置文件，不要硬编码在流程里。
- 每个处理节点都打日志，尤其是“跳过原因”和“LLM 解析失败原因”，否则后期很难排查。
- 为摘要生成保留原文链接，并在推送模板里固定格式，避免 LLM 生成错误链接。
- 把管线拆成三个可独立运行的部分：抓取清洗、摘要过滤、推送。这样某个节点失败时可以手动重跑。
- 先跑一周静默模式，只记录不推送，观察过滤质量和成本，再开推送。

## 总结

RSS + AI 摘要管线的核心不是“把摘要塞给用户”，而是过滤和去重。工程化的关键是：

1. 入口清洗和正文补偿；
2. 结构化输出约束；
3. 增量去重与幂等；
4. 成本与推送频率控制。

这条管线在我这边稳定跑了三周，每天推送从原来的 80+ 条降到 8~12 条，噪音明显下降。即使不用 OpenClaw，用 cron + Python + LLM API 也能复现，逻辑完全一样。如果你已经在用 OpenClaw/Agent/MCP，可以把它作为第一个真正有用的定时自动化任务来练手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d1e20f405181b53d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ec9c0e39ac37714c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d647cb6e159a0da8.png)

