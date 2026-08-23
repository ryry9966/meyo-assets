---
title: RSS + AI 摘要：在 OpenClaw 里搭一条可复现的信息流管线
feedId: 34353
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

我日常订阅 40 多个技术源，每天新增 300+ 条。早期做法是“把 RSS 全文丢给 AI 总结”，结果问题很多：摘要质量不稳定、重复内容多、推送频繁打扰。后来改成一条固定管线：抓取 → 归一化 → 去重过滤 → AI 摘要 → 阈值推送。OpenClaw 只负责调度与查询，AI 只做最后一步摘要。

这条管线跑了几周后，每天真正进入我视野的信息降到 15–20 条，基本没有重复推送。

## 问题

RSS + AI 摘要的难点不在“AI 会不会总结”，而在工程稳定性：

- 源格式混乱：编码、时区、时间字段不统一。
- 重复条目：同一文章在不同源出现，或者 URL 带跟踪参数。
- LLM 输出不稳定：偶尔返回非 JSON、字段缺失、或者开始自由发挥。
- 推送失控：阈值不清晰时，摘要完还是几十条，等于没降噪。

所以需要把每个环节都做成可重试、可校验、可回滚的步骤。

## 做法/步骤

### 1. 抓取与归一化

用 Python + `feedparser` 拉取 RSS。对每个条目做三件事：

- `link` 和 `id` 生成 SHA1，作为去重键。
- `published_parsed` 统一转 UTC，缺失时区时用源默认时区补上。
- 标题和正文做 Unicode 规范化，避免不同编码导致 hash 不一致。

SQLite 建两张表：

```sql
CREATE TABLE seen (hash TEXT PRIMARY KEY, created_at TEXT);
CREATE TABLE summaries (hash TEXT PRIMARY KEY, title TEXT, summary TEXT, score INTEGER, tags TEXT, created_at TEXT);
```

抓到条目后先写 `seen`，即使后续摘要失败也算已处理。这个顺序很关键，否则每次重跑都会重复调用 LLM。

### 2. 过滤与正文提取

不是所有条目都值得进 LLM。先做规则过滤：

- 标题命中黑名单，如“直播预告”“抽奖”“招聘”直接跳过。
- 正文纯文本长度小于 200 字符跳过。
- 同一来源每天最多处理 30 条，避免某个源突然刷屏。

正文提取用 `readability` 或类似库把 HTML 转纯文本。不要全文塞给模型，只保留前 1200 + 后 400 字符，避免 token 爆炸，同时保留结论部分。

### 3. AI 摘要与结构约束

摘要调用 OpenAI-compatible API，强制 JSON 输出。Schema 固定为：

```json
{
  "title": "原标题",
  "summary": "不超过 80 字的中文摘要",
  "tags": ["标签1", "标签2"],
  "action_items": ["可执行事项"],
  "score": 7,
  "skip": false
}
```

Prompt 里写死几条规则：

- 只基于原文，不要补充背景或推断。
- 如果内容是纯公告、活动预告、招聘信息，`skip` 设为 `true`。
- 不要返回 JSON 以外的内容。

工程上必须配合 `response_format={"type": "json_object"}`，再加 Pydantic 校验。校验失败自动重试一次；两次失败写入 `llm_failed`，不阻塞其他条目。

### 4. 接入 OpenClaw

管线本身是一个 CLI 脚本，OpenClaw 的定时任务每小时执行一次。摘要结果写进 SQLite 后，再通过一个轻量 MCP server 暴露给 OpenClaw：

- `search_summaries(query, tags, limit)`：按关键词、标签或时间查询摘要。
- `mark_read(hash)`：标记已读，避免重复推送。

这样做的好处是：OpenClaw 里的 Agent 只负责查询和汇总，不直接碰抓取、去重、摘要逻辑，职责边界清晰。

### 5. 推送

只推送 `score >= 7` 且 `skip=false` 的条目。按来源做每日限额，例如每个源每天最多推 3 条。推送使用 Webhook 通知，不用 Agent 直接发消息，避免幻觉式“自主推送”。

## 踩坑点

- **URL 去重不够**：很多站点 link 带 `utm_source` 等参数。先剥离跟踪参数再生成 hash，否则同一篇文章会重复进库。
- **时区混乱**：部分 RSS 的 `pubDate` 是 GMT，部分没有时区。不要假设服务器时区，用 `dateutil.parser` 解析，缺失时区按源配置补默认值。
- **LLM 输出不稳定**：只靠 prompt 说“返回 JSON”不够。必须 JSON mode + schema 校验 + 重试。温度设低，比如 0.2。
- **摘要失败后的重跑**：如果去重写库放在摘要之后，失败条目每次重跑都会重新调用 LLM。先写 `seen`，摘要失败只记录错误。
- **定时任务限速**：如果一次处理几百条，很容易触发 API 限流。加批处理间隔，例如每批 10 条，间隔 3 秒。

## 可复用建议

- 把管线拆成无状态函数：`fetch`、`normalize`、`filter`、`summarize`、`sink`，方便单独测试。
- 配置放 YAML：源列表、过滤规则、阈值、prompt 模板都放配置文件，不要散在代码里。
- 先跑 dry-run：只输出摘要到本地 JSON，人工抽查 30–50 条，再开启推送。
- 摘要库保留原始 link 和 hash，方便回溯原文，避免 AI 摘要失真。
- 不要把 AI 摘要当成唯一入口，应保留原文链接，关键判断仍可回到源站确认。

## 总结

这套管线的核心不是“用 AI 读 RSS”，而是去重、过滤、格式约束和推送阈值。AI 摘要是最后一步，前面工程化做得越扎实，后面越省心。稳定运行后，每天信息量从 300+ 降到 15–20 条，噪声明显下降，而且几乎不需要人工干预。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/7073901568739a3b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/9d655f158202e69b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/905fbc829cf5c003.png)

