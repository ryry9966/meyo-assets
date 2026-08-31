---
title: RSS + AI 摘要：把订阅噪声压成一条可执行的信号流
feedId: 35520
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

RSS 没死，只是问题从“看不到”变成了“看不完”。一个技术订阅列表每天可能新增上百条，大部分是转载、标题党，或者跟当前关注方向无关。用 AI 摘要不是让它替你读完，而是做第一层分流：哪些值得点开，哪些只需要一句话，哪些直接丢弃。

如果你已经在用 OpenClaw 做定时任务、Agent 编排或 MCP 工具链，这条管线可以挂在现有体系上，不必引入额外后端。核心不是模型有多强，而是把采集、清洗、摘要、分发拆成可回放的小模块。

## 问题

1. RSS 源质量不稳定：标题党、摘要截断、重复转载。
2. 直接整包丢给 LLM，容易上下文溢出、JSON 输出漂移、摘要找不到原文依据。
3. 定时任务失败后难排查，重复消费会浪费 token，甚至污染后续摘要。

## 做法/步骤

### 1. 采集层：先存原始数据

用 `feedparser` 拉取原始 RSS，保存原始 XML/JSON。源站点改版时，有原始数据才能复盘。配置建议用 YAML 管理：

```yaml
sources:
  - name: hn
    url: https://hnrss.org/frontpage
    tags: [tech]
    full_text: false
rules:
  min_content_len: 120
  max_items_per_source: 8
  max_input_tokens_per_item: 2000
```

### 2. 清洗与去重

按 `link` 或标题 + 发布时间做哈希去重。URL 需要先规范化：去掉 `utm_*`、`ref` 等参数。过滤标题黑名单、站点黑名单、过短内容。只保留 `guid`、`link`、`title`、`published` 等必要字段。

### 3. 正文提取

很多 RSS 只给 100 字摘要，直接摘要会失真。用 `trafilatura` 或可读性算法抓原文，超时 8 秒，失败则降级使用 RSS 摘要，并标记为“仅摘要”。不要因为抓取失败就丢掉这条，降级比中断好。

### 4. 结构化摘要

不要让模型自由发挥，给固定 JSON schema：

```json
{
  "title": "原标题",
  "one_line_summary": "不超过 50 字",
  "why_it_matters": "为什么对当前关注方向重要",
  "action": "read|skip|watch",
  "tags": ["ai", "infra"]
}
```

按 tag 分组，每组 5–10 条批量处理，控制每条输入在 1500–2500 token。温度设低，优先用 tool call 或 `json_object` 模式，减少 JSON 格式错误。

### 5. 分发层

按 `action` 分流：`read` 进稍后读或 Notion/飞书多维表，`skip` 只记一行日志，`watch` 进入日历或提醒。通知只发 `read/watch`，不要发摘要墙。

### 6. 调度与容错

OpenClaw 负责定时触发和异常路由。MCP 工具层可以接 RSS 读取、网页抓取；Agent 只在异常处理或判断时介入。每步输出状态文件，支持断点续跑。LLM 调用要 try/catch，失败不阻断整个批次。

## 踩坑点

- **把 RSS 摘要当正文**：先抓原文，或明确标记“仅摘要”，否则摘要会编内容。
- **去重只看 link**：很多站点带动态参数，必须先规范化 URL。
- **不同语种/主题混批**：模型容易跑偏，JSON 输出会忽稳定忽崩。按 tag 分组更稳。
- **JSON 输出失败**：不要只靠 prompt 约束。用函数调用或 `json_object`，设低温度，设定 `max_tokens`。
- **时间窗混乱**：统一 UTC，固定抓最近 24 小时，避免本地时区导致重复或漏抓。
- **抓全文被 ban**：控制并发 1–2，设置真实 UA，间隔 1–2 秒，失败自动降级。

## 可复用建议

- 拆成 `fetch -> extract -> summarize -> deliver` 四个独立步骤，任何一步都能单独重跑。
- 用 `items.jsonl` 追加写每条结果，字段固定，方便回溯和调试。
- 摘要结果必须带原始链接，所有输出只作为提示，不自动执行发布、回复、下单等动作。
- 先跑通 3 个源，稳定两周再扩源。不要第一天接 100 个源，会让排障成本爆炸。
- 设置异常熔断：如果 LLM 输出格式错误率超过阈值，自动停批并保留现场。

## 总结

RSS + AI 摘要的关键不在模型，而在管线约束：去重、原文提取、结构化输出、分流。OpenClaw 适合做调度壳，MCP 适合提供取数/抓取能力，Agent 只处理少数需要判断的环节。这样搭出来的信息流管线是克制的：不替你作决定，只把该看的东西推到面前。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/cc889c0d359e4b96.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/d210e515ac9ae5b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/92de0463aa6b02ef.png)

