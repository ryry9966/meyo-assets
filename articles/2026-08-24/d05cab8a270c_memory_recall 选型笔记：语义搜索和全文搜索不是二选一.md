---
title: memory_recall 选型笔记：语义搜索和全文搜索不是二选一
feedId: 34483
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在给 OpenClaw 的 memory_recall 插件做召回时，起初我直接上了向量库，觉得语义搜索能覆盖用户“说不清楚”的查询。跑了一段时间后发现，在命令、路径、ID、英文报错这类“必须精确”的查询上，向量召回经常把最相关的文档排到很后面，甚至漏掉。后来把全文检索加回来，统一做混合排序，才稳定下来。

## 问题

全文搜索和语义搜索不是在比谁“更智能”，而是处理不同召回类型：

- 全文搜索（FTS/BM25）擅长精确词、专有名词、命令、路径、版本号。
- 语义搜索（embedding/vector）擅长同义改写、模糊意图、跨表达相似。

但 memory_recall 的查询经常混合，例如“上次 ssh 端口改成多少”。这里“ssh”“端口”需要词法匹配，同时“改成多少”又有点口语化。单一后端很难同时做好。

## 做法/步骤

### 1. 拆查询类型

在搜索入口判断查询属于“硬匹配”还是“软匹配”。硬匹配（ID、命令、路径、报错原文、名字）优先全文；软匹配（描述现象、总结需求）走语义。混合模式两类都取，再合并。

### 2. 先做全文保底

本地用 SQLite FTS5 做了最小实现。文档拆成 chunk 后，原始文本写入 FTS 表，同时保留 metadata：source、type、created_at、tags。中文不能直接依赖默认分词，写入前用 jieba 分词，或 SQLite 版本支持时用 trigram tokenizer。短词查询再加 LIKE 兜底。

### 3. 语义召回作为第二路

embedding 模型按预算选，远程 API 效果好但延迟和费用高；本地 sentence-transformers 不花钱，但小模型在专业名词上可能不够。chunk 控制在 300–800 token，太短切碎语义，太长会稀释。向量索引用 sqlite-vec，好处是跟 SQLite 元数据筛在一起，不用额外服务。

### 4. 混合排序

两路召回分数量纲不同，不直接相加。我用 RRF：每个文档分数 `sum(1/(k+rank))`，k 取 60。这样只关心排序，不关心原始分数。

### 5. MCP 暴露

最终把记忆召回包成 MCP 工具：

```json
{
  "query": "redis 6379 连接失败",
  "limit": 5,
  "filters": { "type": "note", "since": "2024-01-01" },
  "mode": "hybrid"
}
```

默认 hybrid，单路模式保留给调试和降级。

## 踩坑点

- SQLite FTS5 中文默认分词很差。不配置 trigram 或预分词，中文基本只能靠字面匹配。
- 向量对数字和命令不敏感。我遇到过一个查询「redis 6379 连接失败」，向量返回的是《Redis 连接超时排查》《Redis 哨兵配置》，真正包含 6379 的配置笔记排到第 7。FTS 一搜就是第 1。
- embedding 模型切换后维度不一致，老索引直接不可用。metadata 里必须记录 model 和 dim，迁移时重建。
- 先向量召回再按 metadata 过滤，会导致候选不足。应该先用 SQL 过滤出候选集，再在候选集内做向量召回。
- 长文档不做 chunk，语义召回到的是整篇，塞进 prompt 很费 token，还容易让模型抓不住重点。
- 低分记忆强行塞给 Agent，等于给上下文注入噪声。设置 score 阈值，宁少勿滥。

## 可复用建议

- 默认先上全文，等出现长尾查询需求再上语义，不要把向量库当默认配置。
- memory_recall 要做成可插拔后端，至少支持 fts、vector、hybrid 三种模式。
- 准备 30–50 条真实查询做评测集，标注理想召回片段，每次调整排序或 chunk 后对比 top-k 召回。
- 对代码、命令、配置类记忆，永远保留原文，不要只存向量。
- 监控每次 recall 的片段 token 数和低分丢弃率，避免“召回越多越好”。

## 总结

语义搜索不是全文搜索的替代，而是对“说不清”的那部分查询的补充。一个能用的 memory_recall，通常是全文保底、语义补充、元数据过滤先行。真正决定质量的往往不是检索算法，而是写入时的 chunk 策略、metadata 完整度和过滤条件。先把这些做扎实，再谈算法选型，会少走很多弯路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/be230edfd37977e5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/8dd3a2b97ba97373.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1f3b9e4da5e46afb.png)

