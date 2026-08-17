---
title: RSS + AI 摘要：在 OpenClaw 上搭一条可停可回滚的信息流管线
feedId: 33605
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

订阅源越多，越容易“抓到但看不完”。RSS 抓取本身不难，难的是把几百条更新变成少量值得行动的信息。直接用 AI 摘要每一篇，常见问题是：摘要太泛、噪声被带进上下文、结果无法触发下一步动作。

这里不讨论“AI 取代阅读”，只分享一条在 OpenClaw 环境里可落地的 RSS + AI 摘要管线。重点放在过滤、结构化输出、失败隔离和可回滚。

## 问题

实际踩到的三个问题：

1. 源质量不一致：大量转载、标题党、重复内容，直接进 AI 会浪费 token。
2. 全文总结效果差：HTML 噪声、导航、广告混进正文，摘要抓不住重点。
3. 摘要没有动作字段：看完不知道“要读、要归档、要建任务”，无法接给 Agent 继续执行。

## 做法 / 步骤

### 1. 统一入口

用 Miniflux 或 RSSHub 做聚合，保留 `canonical_link`、`published_at`、`feed_title`。不要裸抓每个源，否则解析差异会让你在清洗阶段很痛苦。

### 2. 入站清洗与去重

在 `fetch` 和 `summarize` 之间放一层独立清洗：

- 用 `trafilatura` 或 `readability` 提取正文，失败时回退到 meta description。
- 正文超过 4000 字符先截断，但保留原文链接。
- 去重不看 URL，对 `title + 正文前 500 字符` 做 SHA-1。很多站会在 URL 后加参数，URL 去重会漏。
- 规则过滤：域名黑名单、标题正则、最小正文字数。

### 3. 结构化摘要

调用模型时用 JSON schema 或 function calling，不要依赖自然语言输出。Prompt 示例：

```text
你是信息过滤助手。输入是一条 RSS 条目。
输出 JSON：
{
  "title_zh": "中文标题",
  "summary": "不超过80字",
  "entities": ["OpenClaw", "MCP"],
  "relevance": "high|medium|low",
  "action": "read|archive|task",
  "reason": "判断依据"
}
```

把用户关注点作为变量注入，例如：“我关注 MCP 插件、自动化管线、Agent 可靠性”。这样摘要会与个人目标对齐，而不是通用总结。

### 4. 接入 OpenClaw

把上述能力封装成 MCP 工具 `digest_rss`，在 OpenClaw 中注册。Agent 调用后可以按 `action` 执行：

- `read`：继续抓全文，二次摘要。
- `archive`：写入 SQLite 或笔记库。
- `task`：生成待办，进入任务队列。

再暴露一个 `rss_status` 工具，查看最近处理条数、失败源、延迟。

### 5. 推送与归档

我习惯每天生成一个 digest 文件：

```markdown
# 2025-05-20
- [high] OpenClaw MCP 插件设计 | 摘要... | 链接 | read
- [low] 某站点更新公告 | 摘要... | 链接 | archive
```

这种格式可回滚、可 grep，也方便后续手动复查。

## 踩坑点

- **不要用 URL 去重**。同一篇文章可能带不同参数，用内容 hash 更可靠。
- **trafilatura 可能返回空**。一定要有回退策略，否则条目会直接丢失。
- **JSON 输出不稳定**。优先用 `response_format={"type":"json_object"}` 或 function calling，不要用正则抽字段。
- **批量处理容易超时**。每个源单独 try，失败不阻塞整个批次；记录 `feed_id` 后继续。
- **成本控制**。先规则过滤再摘要，设置每日最大条数，例如 200；只有 `high` 条目才抓全文。
- **OpenClaw 上下文不要塞全文**。先给摘要列表，Agent 按需读取全文，避免上下文爆炸。

## 可复用建议

1. 把管线拆成独立阶段：`fetch → normalize → dedupe → filter → summarize → dispatch`。每阶段都支持单独重跑和 dry-run。
2. Prompt 与管线分离，存成 YAML 配置，换模型不改代码。
3. 保留原文链接和正文前几百字。摘要只是索引，不是原文替代。
4. 监控失败率：某个源连续失败 3 次，自动暂停并告警。
5. 时间用 UTC 存储，避免时区导致的重复抓取或漏报。

## 总结

RSS + AI 摘要的重点不是“总结每一篇”，而是把信息流变成可筛选、可触发动作的管线。稳定运行的关键在边界处理：清洗、去重、结构化输出、失败隔离。OpenClaw/MCP 适合做编排层，但重逻辑应放在外层脚本，不要让 Agent 承担所有判断。

---

