---
title: 用 RSS + AI 摘要搭一条自动化信息流管线
feedId: 32443
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么我需要这条管线

信息焦虑不是缺少信息，而是能看到的太多，消化的太少。我订阅了 40 多个技术博客、论文预印本、GitHub trending、Hacker News 和几个行业 newsletter，但真正有时间点进去读完的不超过 5 条/天。大部分时间浪费在“扫标题—点开—发现不感兴趣—关掉”这个循环里。

我试过很多方案：浏览器书签、Inoreader 规则、IFTTT 推送，但最终发现核心问题不是“看不过来”，而是缺乏一个能把原始内容压缩成决策辅助信号的中间层。AI 摘要恰好适合做这个中间层，而 RSS 仍然是互联网上最稳定、最结构化的信源格式。所以决定搭一条管线：从 RSS 中抓取条目，用模型生成关键摘要，再推送到一个轻量级阅读入口。

## 管线设计

整体流程：  
**RSS 源 → 抓取与清洗 → 去重与缓存 → AI 摘要生成 → 输出与分发 → 健康监控**

工具选型上，刻意保持“脚本化”而不是依赖某个平台，方便后续被 OpenClaw、MCP 工具链或任何自动化 Agent 集成。核心组件：

- **抓取**：Python `feedparser`，配合 `requests` 处理重定向和证书错误
- **清洗**：`html2text` 移除样式和脚本，截断过长内容
- **缓存与去重**：SQLite 本地文件，记录 `(feed_url, entry_id, published_parsed)`，避免重复请求和重复摘要
- **AI 摘要**：OpenAI API（也方便替换为本地模型或 Claude API），使用严格的系统提示限制输出格式
- **分发**：生成一个静态 HTML 页面上传到内网服务器，同时在 Terminal 跑一个 `cron` 任务，可选推送到 Telegram bot 作为即时通知
- **监控**：简单的 `healthcheck.io` 回调，搭配错误日志

## 具体步骤

### 1. 抓取与标准化

`feedparser` 能搞定大部分 RSS，但有 20% 的源会让你难受：有的只有 `dc:date` 而缺 `published`，有的把全文放在 `description` 但实际是 HTML 碎片，有的频繁 301 但不告诉你。所以必须加一层规范化函数：

```python
def normalize_entry(entry, feed_url):
    return {
        "id": entry.get("id") or entry.get("link"),
        "title": entry.title.strip(),
        "link": entry.link,
        "published": parse_date(entry),
        "content": extract_content(entry),
        "feed_url": feed_url,
    }
```

`parse_date` 会尝试 `published_parsed`、`updated_parsed`、`dc_date` 字段，并回退到 `datetime.utcnow()`。`extract_content` 取第一个非空的 `content[0].value`、`summary` 或 `description`，然后用 `html2text` 转成纯文本，截断到 6000 字符——防止某些源把全站地图塞进一个条目。

### 2. 去重与增量

SQLite 表结构很简单：

```sql
CREATE TABLE IF NOT EXISTS seen_entries (
    feed_url TEXT,
    entry_id TEXT,
    published TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (feed_url, entry_id)
);
```

每次运行流程：

1. 从所有 RSS 源拉取条目
2. 对每个条目检查 `(feed_url, entry_id)` 是否存在
3. 不存在则插入并送入摘要队列

另外加了一个“过期”策略：不处理发布时间超过 7 天的条目，防止一次性导入旧数据时消耗大量 token。

### 3. AI 摘要

摘要生成是整个管线里最需要控制成本和质量一致性的环节。我测试了三种 prompt 策略，最终落定在 **决策式摘要**：

> “用 3 条以内中文短句总结这篇文章的核心信息。如果是工具发布，点明解决了什么问题、与现有方案的差异；如果是指南类，提取最高频的实操要点；如果难以判断价值，请直接回复‘价值模糊’。”

并强制模型输出格式：  
```
【摘要】...  
【关键词】...  
【评分】1-5 表示信息密度
```

这么做有两个好处：一是摘要可以直接展示而不需要二次剪辑；二是评分字段可以用于后续过滤——我只推送评分 ≥3 的条目，把每日信息量控制在 10 条以内。

API 调用部分加了重试与指数退避，并对 429 做了自动 sleep。成本方面，所有源每天新增条目约 60 条，每条摘要平均录入 ~500 token，输出 ~150 token，每天总消耗约 40k token，gpt-3.5-turbo 下成本很低，用 gpt-4o-mini 以后几乎可以忽略不计。

### 4. 分发与订阅

目前选择最轻的方式：生成一个 `index.html`，按日期降序排列摘要卡片，通过 `rsync` 推送到一个小型 Web 服务器，我在手机上用 Shortcuts 快速打开页面。另外加了一个 `tg_notify` 函数，把评分 4-5 的条目推送 Telegram，消息格式：

```
📌 [评分:4] 标题
摘要内容...
🔗 URL
```

这样即使不主动打开页面，也不会错过极高价值条目。

### 5. 定时与健康检查

`cron` 每天 7:00、12:30、19:00 各运行一次。每次运行末尾会 curl `https://hc-ping.com/<uuid>`，如果超过 2 小时没有任何 ping，healthcheck.io 会发邮件告警。踩过坑：有一次 feedparser 遇到不可解析的 XML 直接抛出异常，整个脚本中断，后面补上了全局 try/except 和 sentry-sdk 的轻量集成。

## 踩坑记录

- **RSS 内容质量方差巨大**：有些源只提供 50 字导语，模型给出摘要也毫无信息量。这种情况直接标记“价值模糊”，并记录低分源，考虑以后移除。
- **字符编码**：遇到一个 `GB2312` 编码的技术博客，`feedparser` 能解析但文本乱码。最终在 `requests.get` 阶段用 `response.apparent_encoding` 探测编码再手动 `.content.decode(...)`，再传给 `feedparser`。
- **链接失效快**：Twitter RSS 桥接的链接经常 48 小时就失效，所以推送时保留原文标题和摘要，不再依赖链接长期有效。
- **摘要的时效性**：对于新闻类源，延迟 6 小时以上就不再有价值，所以分开配置了“高频源”和“日常源”，高频源每小时运行单独脚本，避免消耗过大。

## 可复用建议

如果你也想搭一条类似的管线，可以这样起步：

1. **从 5 个源开始**，而不是马上导入全部订阅。观察不同结构的 RSS 表现，再逐步扩展。
2. **分离抓取和摘要逻辑**：抓取部分每天可以多跑几次，摘要可以合并批处理，省钱且降低 API 错误率。
3. **给摘要加评分机制**：即使不准确，也能作为早期过滤。日后可以把这个评分作为 RLHF 信号去微调一个专门做信息密度评估的小模型。
4. **将脚本封装为 MCP 工具的 Server**：用 MCP 的 `resources/list` 暴露每日摘要列表，`tools/call` 提供“强制重新总结某源”的能力，就可以直接通过 OpenClaw 或其他 Agent 接入，作为知识检索的前置步骤。
5. **保留原始链接**：模型会犯错，当摘要让你感兴趣时，需要一个快速跳回原文的路径。

## 总结

这条管线运行了两个月，实际效果是：我每天阅读订阅源的时间从 35-45 分钟降到 8-10 分钟，错过重要信息的次数显著减少（大概从每周 2 次降到每两周 1 次），而且手机通知不再被标题党攻陷。

它本身不是一个完美的产品，只是把“需要人工判断”那一步尽量后移。对于已经在折腾 OpenClaw、MCP 和 Agent 的用户来说，这种小而明确的自动化管线更适合作为基础设施的一部分，而不是寄望于某个黑盒全自动摘要工具。如果你也在信息过载中挣扎，值得花一个下午把核心流程跑通，后面的收益会不断叠加。

---

