---
title: 用 OpenClaw 搭一条“RSS + AI 摘要”自动化信息流：不追求花活，先解决每天 40 分钟的文章筛选
feedId: 31626
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

做 Agent 开发和信息聚合的朋友，大多有过同样的处境：订阅了 30-60 个 RSS 源，涵盖 GitHub Releases、Hacker News、公司技术博客、arXiv 更新，以及少量独立博客。RSS 本身解决了"信息被抓到一处"的问题，但没解决"信息被读完"的问题——每天堆叠 200+ 条目，一个人根本无法消化。

OpenClaw 的优势在于它是调度中枢，不是另一个信息流 App。配合 MCP 生态里的 HTTP 抓取、数据库存储和通知类工具，你可以在 OpenClaw 内部跑一趟自动化管线，每天晚上帮你把 RSS 里 80% 的噪音过滤掉，只留下 5-10 条值得精读的内容。

## 问题

很多人一开始踩的坑是：直接让 LLM 从零开始抓取和汇总所有 RSS。结果摘要内容平淡无奇，全是"这篇博客讨论了 Kubernetes 的部署之道"之类的废话，AI 味很重，信息量趋近于零。

管线设计需要回答四个具体问题：

1. **上游源**怎么规划？哪些源要全量摘要，哪些源只需要标题扫描？
2. **摘要触发**怎么控制？不能每篇文章都做一次 LLM 调用，成本和时间都撑不住。
3. **输出格式和投递**走哪条路？Telegram/邮件/本地 Markdown 文件，各自适配什么使用场景。
4. **失败恢复**怎么做？某个源挂了或者返回非标准 XML，会不会卡住整条流水线。

## 做法/步骤

### 第一步：明确架构，分开"抓取"与"摘要"

架构核心是把「抓取」和「AI 处理」拆成两个独立阶段。

- **抓取阶段**：用轻量脚本（Python + feedparser，或直接用 OpenClaw 的 HTTP MCP 工具）批量拉取订阅源，把原始 XML 落到本地 JSONL 文件，记录 `feed_url / entry_id / title / link / published / summary` 字段。不要在这个阶段调用任何 LLM。
- **摘要阶段**：读取 JSONL，过滤掉已处理过的 `entry_id`（维护一个 SQLite 或简单 DB），然后按规则分批送入 LLM。

### 第二步：用 OpenClaw 编排定时任务

在 OpenClaw 中定义一条 Cron 任务，建议晚间执行（比如 22:00），流程伪代码如下：

```
load config
fetch feeds -> save to /data/raw/YYYY-MM-DD.jsonl
load processed_ids from state store
filter unprocessed items
if items.empty: notify "今日无新内容" and exit

# 摘要规则分级
priority_items = items where source in (core_blogs, releases) OR title matches /kubernetes|llm|agent/
digest_items = items where source in (news_aggregators) and published within last 24h

for group in chunk(priority_items, size=10):
    call LLM with prompt "extract 3-5 bullet points, include numbers, code names, version numbers, no fluff"
    save to /data/digest/YYYY-MM-DD.md

# 聚合 + 投递
build final digest markdown
send via Telegram bot (or append to Obsidian vault)
```

关键点：**摘要 prompt 一定要要求具体信息**。比如"输出包含具体数字、版本号、项目名的要点列表；如果内容含代码变更，指出变更了什么；不要输出'本文讨论了'这类废话"。回复格式我用 JSON，方便后续二次处理。

### 第三步：优先级分级而非一刀切

把所有源一视同仁做摘要，会把你需要的真正核心信息淹没在无关紧要的新闻里。按信息密度分三档：

- **A 档（全量摘要）**：不超过 5 个源。比如你关注的技术团队博客、某个特定项目的 Release 页面。
- **B 档（标题筛选 + 命中才摘要）**：关键词匹配标题，命中才做全文摘要。
- **C 档（不摘要，只进标题列表）**：行业新闻站、聚合器，人肉扫一眼标题即可。

无论是否摘要，所有条目保留在 `daily_index.md`，保留筛选回溯的余地。

### 第四步：结果输出

我只保留两个出口，避免过度工程：

1. **Telegram Channel**：每天推一条 digest，格式为 `标题 + 摘要 3-5 条 + 链接`。
2. **本地 Markdown 归档**：按日期存到 `digest/2025-xx-xx.md`，方便日后搜索和回看。

## 踩坑点

- **published 字段的时间戳坑**：RSS 里的时间格式极其混乱（RFC 822、RFC 3339、缺失值都有），解析时要统一转成 `datetime` 对象再比较，否则"昨日新增"这个过滤条件会把老文章拉进来，或者把新文章漏掉。
- **摘要输出偶尔膨胀**：同一篇文章反复被摘要（因为 `published` 时间戳是静态的，源站修改后 `updated` 会变化但 `entry_id` 不变）。建议用 `link` 字段做去重键，而不是 `published` 时间。
- **LLM 输出偶尔是 Markdown 表格或无序列表**：我强行把输出类型限定为 JSON 数组，每项 `{title, summary_lines[], link}`，解析时更稳定。
- **HTTP 超时和 SSL 证书问题**：某些源站在国内网络环境下访问极慢或失败。抓取阶段设置单源超时 5 秒，失败自动跳过，绝不阻塞整体流程。
- **中文摘要与英文原文长度比例**：直接翻译会造成信息密度下降。prompt 里写"用原语言提取要点，如果原文为英文摘要用英文输出，禁止中英混排"。
- **费用失控**：不建议用 10 万 token 上限的大模型处理所有条目。A 档文章用强模型，C 档标题过滤不需要模型参与，一切按指令匹配处理。

## 可复用建议

1. **先跑通，再优化**：第一版不用胶囊化和插件化，直接在 OpenClaw 的 workflow 里写顺序步骤。跑两周之后再考虑抽象成插件。
2. **把摘要结果本身当数据源**：每日 digest 写回一个 `digest_archive` 数据库，后续可以叠加做周报、月度趋势统计，这是一条天然的复利路径。
3. **不要过度"自动化"**：OpenClaw 是执行引擎，但筛选判断标准应该始终掌握在你手里。我在 prompt 里刻意保留了"这篇可能不相关"的标注选项，允许模型把拿不准的内容放进"待人工确认"列表，而不是强行归类。
4. **定期审查 RSS 源质量**：每月看一下统计，哪些源累计 30 天 0 命中，直接移除。管线要有自我净化能力。
5. **日志要够细**：把每次运行的条目数、调用 token 数、失败源列表全部落盘。不然你永远不知道某个源已经静默失效了一周。

## 总结

这个管线本质上是"定时任务 + LLM 提取 + 文件落盘 + 消息投递"的组合。OpenClaw 在这次实践里扮演的不是某个不可替代的角色，而是一个可靠的调度底座，不需要写太多胶水代码。

最终收益是：每天起床花 3 分钟看一条 800 字以内的 digest，就能覆盖到前一日最重要的技术动态，精读周期从"攒了一周没看"变成"当天消化"。如果你也在用 RSS 做信息输入，这套方案值得一试。

---

