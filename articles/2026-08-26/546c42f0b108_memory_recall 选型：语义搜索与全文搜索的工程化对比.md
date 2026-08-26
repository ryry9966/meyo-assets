---
title: memory_recall 选型：语义搜索与全文搜索的工程化对比
feedId: 34831
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw/Agent 的 memory 插件或 MCP memory server 中，`memory_recall` 通常负责从长期记忆、历史会话或知识片段中召回相关内容，作为 LLM 的上下文。很多项目默认上向量库做语义搜索，但实际召回质量并不稳定。本文基于本地轻量栈 SQLite FTS5 + sqlite-vec，做一次对比实验。

## 问题

语义搜索（embedding + 向量检索）对同义改写、模糊意图表现好；但对代码 token、日志、UUID、命令、配置名、版本号等精确匹配经常漏。全文搜索（FTS5/BM25）对精确词、低频实体好，但中文分词敏感，且不理解同义表达。

如果只用语义搜索，输入“OOM killed process 1842”可能召回一堆“内存不足”的无关记忆；只用全文搜索，输入“上次那个容器起不来的问题”又可能漏掉“docker daemon restart failed”的历史记录。

## 做法/步骤

1. **准备记忆条目**：每条约 50-300 字，包含 `id`、`content`、`meta(ts, source)`。
2. **建全文索引**：SQLite FTS5。英文直接用 `unicode61`；中文建议显式使用 `trigram` tokenizer，或先 jieba 分词后写入 `content_tokens` 字段再建索引。
3. **建向量索引**：使用 sqlite-vec 或 Chroma。embedding 可选 bge-m3 或 text-embedding-3-small，注意维度固定。
4. **实现双路召回**：同一个 query 同时执行 FTS5 查询和向量 top_k 查询，各取 `top_k=20`。
5. **融合排序**：用 RRF（Reciprocal Rank Fusion）合并两路结果，`score = sum(1/(k + rank))`，`k` 取 60。最终返回 `top_m=5-8`。
6. **记录日志**：记录 query、两路召回 id、RRF 分数、最终是否被上层采纳。

评估阶段构造 60 条查询，分四类：事实型（精确实体/数字）、概念型（同义改写）、时间型、混合型。指标用 recall@5 和 MRR。实测下来：纯全文在事实型 recall@5 约 0.78，纯语义约 0.52；概念型正好相反，纯语义 0.81，纯全文 0.49；混合 RRF 后分别达 0.85 和 0.84。

## 踩坑点

- 中文全文搜索默认 FTS5 `unicode61` 不按词切分，长词匹配经常失败，必须显式使用 `trigram` 或 jieba 预处理。
- 向量搜索对短查询不稳定，尤其只有数字、路径、错误码时，embedding 模型容易退化。建议对长度小于 8 且包含数字/符号的查询直接走全文。
- embedding 模型更换后，旧向量索引不兼容，必须重建，否则召回质量明显下降。
- RRF 的 `k=60` 不是普适值，应针对自己的数据测 20/40/60/80。
- 双路各取 20 条再融合，噪声会增多，最终只给 LLM 5-8 条即可，太多会让模型分心。
- 全文和语义都不考虑时间衰减，需要在召回后加 meta 过滤或时间加权。

## 可复用建议

- 先用全文搜索做 baseline，再决定是否上向量。很多场景 FTS5 已经够用，还省去 embedding 成本。
- 混合优先，但保持查询分类：对精确 ID、路径、错误码走全文；对自然语言问题走语义。
- 本地轻量栈推荐 SQLite FTS5 + sqlite-vec，部署简单、可复现，适合 OpenClaw 插件或 MCP server。
- 建一个 50-100 条的小型回归评估集，每次改检索参数、换 embedding 都跑一遍。
- 所有 recall 接口返回来源和分数，方便调试和排障。

## 总结

`memory_recall` 不是“语义搜索取代全文搜索”的问题，而是根据查询类型和数据特征做混合召回。语义搜索处理同义与模糊，全文搜索处理精确与实体，RRF 做融合是低成本起点。先在本地用 FTS5 + sqlite-vec 跑通并建立评估集，再根据 recall 指标决定是否升级到专业向量库。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/33a2fae50ec1584b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b3c5d024b22ec036.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/42f52235dbd90855.png)

