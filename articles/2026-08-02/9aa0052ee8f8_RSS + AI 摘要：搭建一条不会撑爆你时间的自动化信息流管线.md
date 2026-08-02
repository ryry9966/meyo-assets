---
title: RSS + AI 摘要：搭建一条不会撑爆你时间的自动化信息流管线
feedId: 31299
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：为什么仍在用 RSS，以及为什么还需要 AI

2025 年回过头看，RSS 非但没有死，反而变成重度信息工作者的刚需基础设施。Inoreader、Miniflux、FreshRSS 等自托管方案足够成熟，配合 Telegram/邮箱推送，已经能把散落各处的更新汇聚到一处。

但问题随之而来：订阅源越多，未读计数越像待办清单。每天 200+ 篇文章里，真正需要精读的不到 5 篇，其余只需知道「讲了一件什么事，大概什么结论」就够了。如果每条都点开，信息获取效率反而低于刷几篇高质量 Newsletter。

所以我开始尝试在现有 RSS 管线中插入一层 **AI 摘要**：让模型先读一遍全文，吐出一段 200 字以内的要点，推送到通知频道时直接带上摘要。你扫一眼就知道要不要点开原文，未读积压的心理负担也轻了很多。

这篇文章记录我从零搭建这条管线的完整过程，重点覆盖 **OpenClaw 的 Agent 调度、MCP 插件注入、以及工程落地时容易踩的坑**。如果你已经在用 OpenClaw 做自动化，可以直接把本文的方法挂到现有 Agent 上复用。

## 问题拆解：一条自动化摘要管线需要解决什么

一条合格的「RSS + AI 摘要」管线至少处理四件事：

1. **可靠获取未读条目**：与 RSS 后端 API 交互，拉取指定时间窗口内的文章，能正确处理分页；
2. **内容清洗与截断**：把 HTML 转成可读文本，并限制送入模型的长度，避免 token 浪费；
3. **结构化摘要生成**：调用 LLM 生成固定格式的摘要，同时做好异常重试与限流；
4. **结果分发**：把摘要推送到 Telegram / 企业微信 / 飞书 / Notion 等目标，并标记条目已读（可选）。

用 OpenClaw 搭建时，这四件事会被分配到 **Timer 节点、Function 工具调用链、MCP 连接器** 等不同模块里。好处是每个环节都可以独立调试、替换，不必写一坨面目模糊的 Python 脚本。

## 实战步骤：基于 OpenClaw + Miniflux + MCP 的完整实现

### 1. 准备 RSS 后端

我用的是自部署的 [Miniflux](https://miniflux.app/)，它对 API 的支持极好，Go 语言单二进制，维护成本接近零。安装完成后进 Settings > API Keys 创建一个只读 Key，后面全程要用。

一些关键 API 端点（v1）：

- `GET /v1/entries?status=unread&limit=20&order=published_at` — 取最新未读条目
- `GET /v1/entries/:entryID/fetch-content` — 触发源文抓取（部分源可能返回截断内容）
- `PUT /v1/entries` — 批量标记已读

你可以用 curl 先验证：

```bash
curl -H "X-Auth-Token: YOUR_TOKEN" \
  "https://miniflux.example.com/v1/entries?status=unread&limit=5"
```

### 2. 用 MCP Server 把 Miniflux 变成可调用工具

为了让 OpenClaw Agent 能直接调用 Miniflux 的 API，我写了一个轻量的 MCP server（`mcp-miniflux`），暴露三个 tool：

- `list_unread_entries(limit, older_than_days)` → 取回未读条目列表
- `get_entry_content(entry_id)` → 返回清洗后的纯文本内容
- `mark_entries_read(entry_ids)` → 批量标记已读

在 OpenClaw 的 Agent 配置中通过 mcpServers 引入即可：

```json
"mcpServers": {
  "miniflux": {
    "command": "node",
    "args": ["./mcp-miniflux/dist/index.js"],
    "env": {
      "MINIFLUX_URL": "https://miniflux.example.com",
      "MINIFLUX_API_KEY": "YOUR_KEY"
    }
  }
}
```

这样一来，Agent 在规划步骤时，就可以自然地调用 `list_unread_entries` 获取文章列表，再对每一篇调用 `get_entry_content` 拿正文。完全不需要硬编码 HTTP 请求。

### 3. 设计摘要 Prompt & 调用 LLM

摘要生成我用的是 OpenClaw 内置的 `llm_call` 节点（底层可指向任何兼容 OpenAI 接口的服务，本地模型也可以）。Prompt 设计要点：

- 明确输出 JSON 结构，例如 `{"summary":"...","takeaway":"...","worth_reading": true/false}`
- 限制摘要长度，直接用 `max_tokens` 压到 150-200
- 强调「只提取事实和结论，不做二次演绎」

示例 system prompt：

```
你是一个信息助理。用户会给你一篇博文/新闻全文，请返回 JSON：
- summary: 用 3 句话概括文章内容，不含评价
- takeaway: 对技术/行业读者的 1 条可操作启发，如果没有则写 "none"
- worth_reading: 如果文章包含具体方法、数据或独特观点，为 true，否则 false

只输出 JSON，不要其他文字。
```

### 4. 组装成 OpenClaw 定时任务

在 OpenClaw 中创建一个 `CronAgent`，每 30 分钟触发一次：

1. 调用 `list_unread_entries(limit=20, older_than_days=1)`
2. 对每条 entry，调用 `get_entry_content` 获取内容；若正文为空，用 `entry.title` 作为 fallback
3. 对每篇文章调用 LLM 获取摘要 JSON
4. 把结果拼成 Markdown 消息，通过 `telegram_send_message` 或 `webhook` 推送
5. 可选：调用 `mark_entries_read` 批量标记已读

OpenClaw 的流程编排是树状的，上面每一步失败都可以设置 retry 分支或 fallback，不需要额外写胶水代码。

## 踩坑记录与生产级建议

这里列几个最容易栽跟头的地方，很多不是 API 的问题，而是「工程感觉」的点。

**HTML 到纯文本的清洗不彻底**  
很多 RSS 源的 `content` 字段是完整 HTML，直接喂给 LLM 会夹杂大量广告、导航、评论区噪声。MCP server 里我用了 `turndown` 转 Markdown 后再用正则切掉导航和 footer，摘要质量明显提升。如果为了省 token，可以对极长文章截断到 4000 字符。

**摘要生成不能并行无脑怼**  
如果一次拉取 20 条未读，20 次并发 LLM 调用可能打爆 rate limit。OpenClaw 的 `concurrencyLimit` 参数在这里很有用，建议设为 3-5，配合指数退避重试，既稳定又不会太慢。

**分页陷阱**  
Miniflux 默认 limit 上限 100，如果单次拉取超过未读数，实际上不会有 next page。但如果你设置了 `older_than_days=1`，同时又大量积压，可能会漏掉。最佳实践是把定时任务跑勤一点（15-30 min），每次只处理最新 20-30 条，自然削峰。

**标记已读的时机要掂量**  
如果摘要推送到 Telegram 后立刻标记已读，一旦通知丢失，那篇文章就可能永远错过。我选择「延迟标记已读」：推送到一个「待办频道」后不立即标记，而是在第二天统一清理。这个逻辑在 OpenClaw 里可以拆成两个 Agent，一个负责摘要推送，一个负责清理已读，互不干扰。

**本地模型与 token 成本**  
如果每天处理 200+ 篇文章，即使使用 gpt-4o-mini 也很快烧完免费额度。我最后把摘要任务迁到了本机跑 Qwen2.5-7B-Instruct（通过 Ollama 暴露），单次摘要耗时 ~3-5 秒，成本为零。OpenClaw 的 `llm_call` 节点改一下 `base_url` 即可无缝切换。

## 可复用建议：用 MCP + Agent 思维替代脚本思维

这个管线的真正价值不在于「又一个自动摘要机器人」，而在于 **把信息处理流程抽象成可组合的 Agent 工具链**。

- **如果你用 OpenClaw**：上面的 MCP server 和 Agent 模板可以直接套用到其他 RSS 后端（FreshRSS 等），只需要换 API 适配层；
- **如果你用其他 Agent 框架**：MCP 的 tool 定义可以原样迁移，配合任意 Function Calling 能力实现同样效果；
- **如果只想轻量复用**：把 Miniflux → LLM 部分抽成一个独立 Python/Node 脚本，用 cron 调度，照样能跑，只是少了内置重试、限流、步骤可视化这些工程化便利。

另外，这套管线还可以横向扩展：比如接入 `web_scrape` MCP tool 对原文链接做更深度解析（适合仅提供摘要的 RSS），或者结合 `vector_search` 工具把摘要存入本地知识库供日后检索——但那就是另一个话题了。

## 总结

RSS + AI 摘要的核心不是「省去阅读」，而是 **快速判断哪些值得读**。用 OpenClaw 的 Agent 调度能力搭配 MCP server，能在不写过多胶水代码的前提下，获得一条稳定、低维护成本、可组合的个人信息管线。

最后几点工程原则：

- 先跑通 5 条的单步手动触发，再上定时任务
- LLM 摘要的 JSON 输出结构尽量稳定，方便后续做筛选或统计
- 不要把标记已读和推送绑得太紧，容错比效率更重要

这条管线跑了三周，我的 RSS 未读焦虑基本归零。每天早晚各扫一眼 Telegram 摘要流，需要细读的文章点开，不需要的直接忽略或归档，信息摄入密度反而更高了。

---

