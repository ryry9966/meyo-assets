---
title: Memory Recall 选型：语义搜索与全文搜索的工程对比
feedId: 33681
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw 这类 Agent 里做 memory recall，常见做法是把用户偏好、历史决策、报错信息存进向量库，查询时用 embedding 做语义搜索。这个方案在自然语言问题上表现不错，但落到自动化、MCP 工具链里，容易出问题：精确命令、ID、报错码、版本号这类“硬事实”经常召回不准确。于是有人转向全文搜索。问题变成：语义搜索和全文搜索到底怎么选？

## 问题

语义搜索擅长同义改写，比如“数据库锁了怎么办”能匹配“sqlite lock timeout”，但在短文本、错误码、路径、英文缩写上容易漂移。全文搜索擅长精确 token 匹配，比如报错 `database is locked`、工具名 `openclaw_mcp`、配置项 `max_retries`，但对自然语言表达和上下文意图不敏感。Agent 的 memory recall 往往同时需要这两种能力。

## 做法

我建议不要二选一，而是做双通道召回，再用 RRF（Reciprocal Rank Fusion）合并。具体步骤：

1. 统一 memory item schema：`id`、`content`、`type`（preference/error/decision/tool_use）、`tags`、`created_at`、`last_access_at`、`source`。
2. 全文通道：用 SQLite FTS5 建表，`content` 列加索引。如果中英混合，可以加 trigram tokenizer，或写入前用 jieba 分词做额外列。
3. 语义通道：`content` 生成 embedding，存 sqlite-vec 或轻量向量库。模型选本地小模型即可，避免外部依赖。
4. 写入时双写：先写全文索引，再写向量索引。全文失败可以降级，语义失败不影响主流程。
5. 查询时并行召回：两个通道各取 `top_k=10`，然后用 RRF 公式合并：`score = sum(1/(k + rank))`，`k` 取 60。最后按 `type` 和 `recency` 做轻量加权。
6. 对 ID、命令、错误码这种字段，直接走 exact match 或 FTS 的 phrase query，不要依赖语义。

## 踩坑点

- **语义搜索阈值很难调**：阈值高漏召回，阈值低噪声多。不要单独靠 similarity 阈值决定是否采用，而是把阈值当弱信号。
- **FTS 对中文默认分词差**：SQLite FTS5 默认 unicode61 对中文基本按整句处理，搜“数据库”可能失配。要么用 trigram，要么写入前分词。
- **短文本 embedding 不敏感**：单独存 `max_retries=3` 或 `openclaw_mcp` 语义召回容易乱，应该给这类内容打 `type` 并优先全文。
- **记忆会过期**：报错信息、临时决策如果不加 TTL 或 `last_access_at` 衰减，会污染召回。建议每次 recall 命中时更新 `last_access_at`，并定期清理低价值记忆。
- **双通道性能**：并行查两个索引可以，但合并后单次 recall 仍可能超过 Agent 上下文。截断时优先保留全文命中和最近使用记忆。
- **评测缺失**：很多人只凭感觉调。至少准备 30 条 query，覆盖报错、命令、英文缩写、中文同义改写、模糊意图。

## 可复用建议

先上全文搜索做底线，再上语义搜索做增强；不要一上来就向量库。对固定 token 类内容，强制走 exact/FTS。召回结果带上 `source` 和 `type`，方便给 LLM 排序和过滤。把 recall 未命中、误命中记录到日志，每周看一次。如果你在 OpenClaw 插件里做 memory，MCP server 的 `resource/list` 返回也可以保留一份全文索引，别只暴露 vector search 接口。

## 总结

语义搜索和全文搜索不是替代关系。Agent 的 memory recall 里，语义负责模糊意图，全文负责精确事实。工程上最稳的做法是双通道 + RRF + 规则兜底。先保证不会漏掉错误码和命令，再去优化自然语言召回。这个顺序比盲目上向量库重要得多。

---

