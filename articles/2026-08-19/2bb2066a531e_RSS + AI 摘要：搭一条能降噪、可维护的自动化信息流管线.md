---
title: RSS + AI 摘要：搭一条能降噪、可维护的自动化信息流管线
feedId: 33843
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

订阅源多了以后，问题不是“看不到信息”，而是“看不完、看不动”。我的 RSS 阅读器里长期堆着 60+ 个源，每天新增几百条，真正值得点开的不超过 20 条。于是开始做一条自动化管线：RSS 负责采集，AI 负责压缩摘要，OpenClaw 负责调度和推送。

目标很明确：**不追求全自动读懂一切，只做“这条信息值不值得花时间”的决策辅助。**

## 问题

直接让 LLM 读 RSS 原始条目会遇到几个工程问题：

- RSS `<description>` 经常是截断的 HTML，摘要质量不稳定。
- 全文抓取失败率高：登录墙、反爬、懒加载、编码问题。
- 重复处理：同一个链接可能因为 GUID 不稳定或源重发被摘要多次。
- LLM 成本不可控：全文直接灌入大模型，token 消耗很快。
- 推送杂音：如果每条都推，只是把信息过载从阅读器搬到 IM。

## 做法/步骤

整个管线分五层，OpenClaw 作为中间调度。

### 1. 采集层：统一 RSS 源

自建 FreshRSS 或 Miniflux 作为订阅后端；没有原生 RSS 的站点用 RSSHub 生成 feed。OpenClaw 不需要直接面向几十个源，只需要消费一个统一的输出。

### 2. 工具层：封装成 MCP 工具

把抓取和解析能力封装成 MCP 工具，方便 Agent 调用：

- `rss_unread_items`：读取未处理的 RSS 条目。
- `extract_article`：用 trafilatura 或 Readability 抽取正文。
- `save_digest_state`：写状态库。

这样可以避免在 Agent prompt 里塞一堆抓取逻辑。

### 3. 调度层：OpenClaw 定时任务

在 OpenClaw 侧挂一个 cron 任务，每 30 分钟触发一次 `rss-digest` Agent。任务流程固定：

1. 获取未处理条目，按源权重和发布时间排序。
2. 对每条执行 `extract_article`，失败则降级使用 RSS `description`。
3. 截断正文到 4000-6000 字符，调用 LLM 按固定 JSON 输出摘要。
4. 写入状态库，标记 `fetched -> summarized -> pushed`。

摘要 prompt 尽量小：

```text
请判断以下文章对 AI 自动化/工程实践是否值得阅读。
输出 JSON：
{
  "title": "...",
  "summary": "不超过80字中文摘要",
  "tech_keywords": ["..."],
  "worth_reading": true/false,
  "reason": "一句话理由"
}
```

### 4. 状态层：幂等与去重

用 SQLite 表维护状态，唯一键为 `link_hash`（对规范化后的 URL 做 SHA-256）。即使 GUID 变化，只要 URL 相同就不会重复处理。表结构：

```sql
CREATE TABLE items (
  link_hash TEXT PRIMARY KEY,
  source TEXT,
  title TEXT,
  link TEXT,
  published_at TEXT,
  status TEXT,
  summary TEXT,
  created_at TEXT
);
```

### 5. 推送层：降噪后输出

每轮只聚合成一条 digest，最多 10 条，通过 OpenClaw 的 notifier 插件推送到 Telegram 或企业微信。推送间隔设最小 2 小时，避免频繁打扰。

## 踩坑点

- **RSS 编码**：用现成解析库，不要手写正则。
- **相对链接**：正文里的 `<img src="/foo.png">` 要还原为绝对 URL，否则摘要里图片全挂。
- **登录墙**：抓取失败直接放弃，不要反复重试浪费资源。
- **JSON 稳定性**：LLM 偶尔输出非法 JSON。优先用 function calling 或 JSON mode，并加一次解析失败重试。
- **并发重复**：多个 OpenClaw 实例可能同时跑同一 cron。给任务加文件锁或数据库锁，保证单实例执行。
- **标题党源**：对低质量源做权重降级，或者只提取不推送摘要。

## 可复用建议

- **统一 schema**：摘要层统一输出 `source / title / link / summary / tech_keywords / worth_reading / reason`，方便后续消费。
- **先跑 5-10 个高质量源**，稳定后再扩展，不要一上来接几十个源。
- **保留原文链接和原始条目**，不要用 AI 摘要替代原文，它只是为了筛选。
- **做好 token 预算**：先用便宜模型做初筛，只对高价值条目用强模型做细摘要。
- **监控队列长度和失败率**：如果 `extract_article` 成功率低于 60%，先检查抓取工具而不是换 LLM。

## 总结

这条管线实际运行下来，最大的收益不是“读了更多”，而是减少了无效点击和稍后读堆积。RSS 负责稳定采集，LLM 负责可丢弃的摘要压缩，OpenClaw 负责把工具和定时任务粘在一起。真正决定系统可维护性的，是状态管理、幂等处理和降级策略，而不是 AI 模型多强。

如果你已经在用 OpenClaw/Agent/MCP，这套方案大约一个晚上就能搭完骨架；后续大部分时间会花在源质量筛选和 prompt 微调上。

---

