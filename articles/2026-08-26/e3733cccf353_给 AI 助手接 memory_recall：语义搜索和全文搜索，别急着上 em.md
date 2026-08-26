---
title: 给 AI 助手接 memory_recall：语义搜索和全文搜索，别急着上 embedding
feedId: 34797
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

给 OpenClaw 这类 Agent 做记忆召回时，常见需求是：用户说“上次那个报错怎么修的”，助手能从历史会话、笔记、运行日志里捞出相关片段。memory_recall 工具本质是一个检索问题。方案通常分两类：

- 全文搜索：SQLite FTS5、BM25、关键词匹配；
- 语义搜索：embedding + 向量相似度。

很多人上来就 embedding，觉得更“智能”。但我在多个场景里发现，纯向量召回经常漏掉强特征查询。

## 问题

语义搜索擅长同义改写，比如“怎么让程序更省内存”能关联到“降低 RAM 占用”。但 Agent 实际查询里，大量 query 是强特征：路径、报错码、命令名、版本号、配置项。例如：

- `CUDA OOM`、`/tmp/xxx.log`
- `pydantic v2`、`sqlite-vec`
- `memory_recall timeout`

这些查询，全文搜索几乎必中；语义搜索可能因为 embedding 模型没见过这些 token，或它们被拆成子词后向量漂移，导致召回一堆相似但不相关的聊天。

反过来，全文搜索对“我上次怎么解决的那个模型加载慢的问题”这种口语化、零关键词重叠的 query 不友好。

所以问题不是“哪个好”，而是：你的 memory 数据里，强特征查询占比多少？系统是否有元数据过滤、时间衰减？数据量多大？

## 做法/步骤

我在 OpenClaw 的 memory 模块里先做了一套可回退方案：

1. 存储层用 SQLite FTS5 建全文索引。表结构至少保留 `id, source, memory_type, content, created_at, updated_at`。内容追加写入，保留原始文本。

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  source,
  memory_type,
  tokenize='unicode61'
);
```

2. 如果中文场景多，把 tokenizer 换成 `trigram` 或外部 jieba 分词；否则长中文会被整段切，召回率很差。

3. 先跑全文搜索，取 `bm25(memory_fts)` 排序，限制 top 20 作为候选。

4. 语义召回作为可选增强，而不是主路径。对 `content` 做 256-512 token 的 chunk（重叠 50-100），生成 embedding 存 sqlite-vec。query 先走 FTS 拿候选，再用向量相似度重排；或直接用 RRF 融合两路结果。

5. 给 memory_recall 工具加过滤参数：`memory_type`、`start_time`、`end_time`、`limit`、`min_score`。时间过滤能避免“旧记忆永远霸榜”。

6. 设置评估集：固定 30-50 条真实 query，标记该召回的 chunk。对比 `hit@5`、`mrr`。这是唯一能判断“哪个好”的方法，不要靠感觉。

## 踩坑点

- 纯向量搜索对短 query 和专有名词很不稳定。如果你查 `CUDA OOM`，向量可能返回“内存不足”相关但不包含 CUDA 的内容。
- FTS5 默认 tokenizer 中文不友好。用 `unicode61` 基本按空格/标点切，中文长句会变成一整个 token，召回差。
- 向量召回如果没有阈值，可能强行返回相似度只有 0.5 的噪声。要设置 `min_score`，或至少让模型给出明确“无相关”判断。
- chunk 过大，语义被其他内容淹没；chunk 过小，上下文丢失。建议从 256-512 token 试起。
- 只做语义不做全文，Debug 时很难解释“为什么召回这条”。全文检索可解释，便于排查。
- embedding 模型和查询域要匹配。通用模型对代码、日志、命令的表示不如专门增量微调后的模型。如果数据里大量代码，可以对代码抽关键 token 后再向量化。

## 可复用建议

- 先用 FTS5 覆盖 80% 强特征查询，再按需补语义。
- 少于 1 万条数据不必上向量库，`LIKE` + FTS5 足够。
- 大规模、多语言、多模态再引入 embedding 和重排，别提前优化。
- 把 `memory_recall` 设计成可插拔：retriever 接口，支持 `fts_only`、`vector_only`、`hybrid`，方便线上切换。
- 记录每次召回的 `source` 和 `created_at`，让 Agent 知道“这是来自哪次会话、什么时候的记忆”，比单纯给文本更重要。
- 做时间衰减：新记忆优先，旧记忆可降权，但不要完全忽略，否则旧知识丢失。

## 总结

语义搜索和全文搜索不是替代关系。工程上更推荐：全文检索托底 + 语义检索增强 + 元数据过滤 + 评估集验证。尤其给 Agent 做 memory_recall，可解释性和稳定性比“看起来更智能”重要得多。先用最朴素的 FTS5 跑通闭环，再根据真实 query 的失败案例逐步加向量检索，成本最低、收益最明确。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/1bb2f03718ba9db6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/ab1e78f975091e18.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/d34fceaf9603fcdf.png)

