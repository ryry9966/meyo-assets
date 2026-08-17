---
title: memory_recall 选型：语义搜索与全文搜索不是二选一
feedId: 33610
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 的 Agent 实践里，memory_recall 经常被简化成“把历史消息丢进向量库，然后 top_k 召回”。但当用户问“上个月那个 500 错误的排查结论”或者“帮我找包含订单号 ORD-20240213 的记录”时，纯语义搜索很容易翻车。问题不是 embedding 不行，而是召回目标不同：语义搜索找“意思相近”，全文搜索找“字符/词项一致”。

## 问题

memory 召回通常有三类需求：

1. **概念性回忆**：例如“之前怎么解决 token 超限的？”
2. **精确匹配**：订单号、错误码、函数名、日期、版本号。
3. **条件过滤**：某个项目、某个用户、最近 N 天。

纯语义搜索对第 1 类友好，对第 2 类不稳定；纯全文搜索对第 2 类友好，但对同义词、改写、跨语言表达很弱。实际工程里很难靠单一方案覆盖。

## 做法/步骤

我在 OpenClaw 的 memory_recall 插件里改成混合召回，结构如下：

**写入时**：每条 memory 同时维护两套索引。

- SQLite FTS5 建全文索引，字段包含 `content`、`tags`、`source`。
- 向量库存 embedding，metadata 保留 `ts`、`project`、`conversation_id`、`type`。

**召回时**：同一个 query 并行发两个检索。

- FTS5 用 BM25 取 top 20。
- 向量用 cosine 取 top 20。

**融合**：用 RRF（Reciprocal Rank Fusion）合并两路结果，`k` 取 60。RRF 的好处是不用关心 BM25 和余弦分数的尺度差异，比线性加权稳定。

**过滤**：结构化条件在查询前下推，例如 `project = 'foo' AND ts > date('now','-30 day')`。不要先召回再 filter，否则容易丢候选。

**重排**：如果候选集不大，再过一个 cross-encoder reranker；日常内存召回可以先不上，看延迟预算。

## 踩坑点

1. **Embedding 对 ID、代码、日期不敏感**。`ORD-20240213` 和 `ORD-20240214` 在向量空间里可能很近，但业务上是两个完全不同的对象。这类信息必须走全文/精确索引。
2. **FTS5 默认分词对中文不友好**。需要接入 trigram tokenizer 或 jieba 分词，否则“语义搜索”查不出“语义检索”。
3. **混合权重不要写死**。不同 memory 分布下，BM25 和向量分数方差很大。线性融合前先用验证集扫几组权重；如果不想调参，直接上 RRF。
4. **Chunk 大小同时影响两路召回**。过大 chunk 会稀释语义，过小 chunk 让全文匹配失去上下文。memory 场景建议 300–800 token 一段。
5. **只做 top_k 不过滤 metadata 会召回过期内容**。时间衰减和 project 过滤应作为硬条件，而不是让模型自己判断。

## 可复用建议

- **先上 FTS5 全文检索做 baseline，再加语义召回**。对多数工具型 Agent，精确匹配和关键词召回能先解决 80% 的问题。
- **把 memory 字段分层**：`content` 走混合召回，`metadata` 走 SQL/过滤。不要把 metadata 拼进 embedding。
- **建小型 eval 集**：20–30 条 query，标注相关 memory id，计算 `recall@5`、`recall@10`。每次换 embedding 模型或融合参数时跑一遍。
- **memory 量小于 10k 时，不必上重型向量库**。SQLite FTS5 + sqlite-vec 足够。
- **总结型 memory 和原文型 memory 分开**。总结适合语义搜索，原文适合全文搜索和引用。

## 总结

memory_recall 没有绝对优劣。语义搜索解决“想不起原话但知道意思”，全文搜索解决“知道确切标识但不想翻聊天记录”。更工程化的做法是混合召回 + 结构化过滤 + 小规模 eval，而不是追求某个单一检索方案。下一步可以加时间衰减、按项目分库，或记录失败召回，逐步把 memory 从“能搜”变成“搜得准”。

---

