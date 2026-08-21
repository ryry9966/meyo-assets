---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索，别只选一边
feedId: 34056
source: 综合讨论
publishedAt: 2026-08-21
---

在给 OpenClaw 写 memory 插件时，最容易卡的环节不是 embedding，而是 `memory_recall` 的召回质量。团队里常有两种声音：一种认为全文搜索足够精确，另一种认为向量语义搜索更聪明。实际跑过一段后，我的结论是：`memory_recall` 不该二选一，而应让两者各管一段。

## 背景

`memory_recall` 的作用是从历史记忆里找出与当前上下文最相关的条目。但 OpenClaw 里的记忆内容通常不是单一类型，而是混合的：

- 用户偏好：`“不要凌晨跑批处理任务”`
- 项目路径：`/data/apps/crawler`
- 错误签名：`TimeoutError: connect 10.0.0.3:5432`
- 决策理由：`“上次没有切 Redis，因为容器内存不够”`

这些内容分布完全不同。偏好和决策理由偏自然语言，路径、IP、错误签名偏精确实体。单一检索方式必然有盲区。

## 问题

全文搜索对错误签名、路径、ID、命令非常稳，但用户换个问法“上次那个数据库连不上是什么原因”，关键词完全对不上。

语义搜索能理解“数据库连不上”约等于“TimeoutError connect”，但对版本号、具体 IP、代码片段不敏感。embedding 模型通常把数字和精确字符串当作弱特征，很难稳定命中。

所以在 `memory_recall` 里，真正需要的是混合检索，而不是纠结“哪个更好”。

## 做法 / 步骤

先定义 memory 的结构：

```text
id, session_id, type(preference|task|error|decision), content,
tags, importance, created_at, updated_at, embedding, fts_content
```

写入路径可以这样设计：

```text
memory_add
  -> 原文写入 content
  -> 分词/清洗后写入 fts_content
  -> 摘要或短句 embedding 后写入向量库
```

召回路径：

```text
query
  -> fts_search(topK=5)
  -> vector_search(topK=10)
  -> RRF merge
  -> filter(importance >= 0.3, recency_boost)
  -> return top 3-5
```

全文索引建议用 SQLite FTS5，轻量、适合 OpenClaw 插件单机场景。中文需要先处理，可以用 jieba 预分词后以空格连接写入 `fts_content`，例如：

```text
数据库 连接 超时
```

语义索引可以用 Chroma 或 pgvector。embedding 模型选 `bge-small-zh` 或 `text-embedding-3-small`，维度 512 或 1536。不要直接对长 `content` 做向量化，先截断到 256-512 token，或者生成摘要再向量化。

合并阶段用 RRF，而不是直接加权。公式很简单：

```text
score(d) = Σ 1 / (k + rank_i)
```

`k` 一般取 60。这样只看排名，不受分数量纲影响。embedding 相似度通常集中在 0.7-0.9，BM25 分数可能是 5-15，直接相加会让全文搜索结果永远沉底。

## 踩坑点

1. **中文分词问题**  
   SQLite FTS5 默认分词器对中文不友好，容易切成单字或整块。必须预分词，否则全文搜索基本不可用。

2. **直接向量化长文**  
   把 2000 字的 task 日志直接塞进 embedding，相似度会趋于平庸。只对摘要或前几百 token 做向量化更稳定。

3. **分数不归一**  
   如果不用 RRF，两个分数的量级差异会直接淘汰全文搜索结果。调参时先看每条结果的来源，确认 fts 和 vector 是否都有贡献。

4. **精确实体被语义稀释**  
   IP、路径、命令、版本号这些内容要完整保留在原文和全文索引里，不要为了向量化而转义或摘要，否则精确召回会丢失。

5. **插件生命周期与连接**  
   在 OpenClaw 插件里接 Chroma/Qdrant 时，注意 client 的创建和释放，避免每次调用都重新初始化。embedding 请求最好加缓存，减少重复调用模型。

## 可复用建议

- **小于 1 万条记忆**：SQLite FTS5 + 内存向量搜索足够，不必上独立向量库。
- **1 万到 50 万条**：SQLite 存原文，Chroma 或 Qdrant 存向量，双写。
- **更大规模或多租户**：Elasticsearch 的 kNN + BM25 混合查询有原生支持，可以减少自研 RRF。
- `memory_recall` 的返回值应带 `source` 字段，标识每条来源是 `fts`、`vector` 还是 `rrf`，并附分数，方便调参。

## 总结

语义搜索适合模糊联想和自然语言改写，全文搜索适合精确实体与代码。OpenClaw 场景里，`memory_recall` 最好的形态是混合检索：全文保底、语义扩展、规则重排。不要用“哪个更好”的思路去替换，而把它们当作两个可随时降级的召回源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/18f344a2f7273f4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b741d0f0bcc4f825.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/e59b02d29f86e7af.png)

