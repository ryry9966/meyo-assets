---
title: 语义搜索 vs 全文搜索：Agent memory_recall 的工程选型与避坑
feedId: 31317
source: 综合讨论
publishedAt: 2026-08-02
---

## 1. 背景：memory_recall 在 Agent 中的角色

无论你用 OpenClaw 的 memory 插件，还是自己给 Agent 搭记忆层，`memory_recall` 都是核心模块——它负责从历史会话、知识片段、工具结果中捞出最相关的内容，拼进下一轮 prompt。常见的检索后端不外乎两类：

- **全文搜索（lexical search）**：基于关键词匹配，例如 BM25、pg_trgm、Elasticsearch。
- **语义搜索（semantic search）**：将文本编码成向量，用余弦相似度或近似最近邻（ANN）检索，例如 pgvector + text-embedding-3-small。

帖子里常有人把语义搜索捧成“下一代搜索”，但工程落地时你会发现，纯向量召回很可能把一些明显的关键词硬匹配都漏掉。于是我们不得不回头审视：对于 memory recall 这种混合了自然语言、代码片段、实体 ID 的场景，到底该选哪种，又该怎样组合。

## 2. 问题拆解：两种搜索的优劣边界

先看真实的行为差异。假设 Agent 记忆库中有这样一条：

> 用户报告了 BUG-1420，问题出在 payment-service 的 retry 逻辑。

| 查询 | 全文搜索行为 | 语义搜索行为 |
|------|--------------|--------------|
| “BUG-1420” | ✅ 精准命中 | ❌ 向量模型可能从未见过这个 ID，召回排名靠后 |
| “支付服务重试失败” | ❌ 完全没有“payment-service”这个英文词，可能零召回 | ✅ 能理解“支付服务” ≈ “payment-service”，“重试” ≈ “retry” |
| “java.lang.NullPointerException 在哪个服务” | ✅ 对异常类名、代码符号有天然匹配能力 | ❌ 除非微调，否则这些特殊 token 的语义很弱 |
| “上次那个支付的问题” | ❌ 模糊，依赖分词运气 | ✅ 能跨句理解指代 |

结论很直白：**全文搜索精于“硬匹配”，语义搜索胜在“泛化理解”。** Agent 既会查具体的 ID、命令、报错栈，又会问“前几天那个小票打印的问题”，你很难单靠一种召回满足所有。

## 3. 做法：构建可切换的 memory_recall

我们自己在 OpenClaw 环境里把 memory 召回抽象成了可插拔的 retriever，通过一个配置项切换后端，便于对比评测。核心步骤：

1. **统一存储结构**  
   所有记忆片段存入 Postgres，`memory` 表包含 `id`, `content`, `metadata`（来源、时间戳等），并建立全文索引和向量列。
   ```sql
   -- 全文索引（pg_trgm + GIN，适合短文本）
   CREATE INDEX idx_memory_trgm ON memory USING GIN (content gin_trgm_ops);
   -- 向量列（1536 维，openai embedding）
   ALTER TABLE memory ADD COLUMN embedding vector(1536);
   ```

2. **向量生成与索引**  
   在写入记忆时调用嵌入服务（OpenAI/text-embedding-3-small 或本地 BGE），写入 `embedding` 列。查询时生成查询向量，用 IVFFlat 或 HNSW 做 ANN：
   ```sql
   SELECT content, 1 - (embedding <=> query_embedding) AS similarity
   FROM memory ORDER BY embedding <=> query_embedding LIMIT 10;
   ```

3. **全文检索查询**  
   对于用户查询，进行关键词提取后生成 `ts_query`，结合 `ts_rank`：
   ```sql
   SELECT content, ts_rank(to_tsvector('english', content), query) AS rank
   FROM memory WHERE to_tsvector('english', content) @@ query
   ORDER BY rank DESC LIMIT 10;
   ```
   如果中英混合，需要对应词典配置，或者直接用 `pg_trgm` 的 `similarity()` 函数，容忍拼写差异。

4. **混合检索与重排序**  
   我们把两种召回的结果取并集（各 top10），然后用一个轻量 reranker（如 bge-reranker-v2-m3 或 cross-encoder）对候选集统一打分，返回 top-K。

## 4. 踩坑记录

- **嵌入模型对特定实体盲区**  
  即使号称多语言的模型，对“BUG-1420”、“cron:0 30 2 * * ?”这类 token 几乎生成相同方向向量，导致相似度无法区分。单纯语义搜索会把这些精确匹配吞没。**解决**：加入关键词过滤器，若查询中包含明确 ID、代码片段，强制使用全文分支，或提升其权重。

- **语言混杂导致全文分词失效**  
  Postgres 默认 `english` 词典对中文完全无效，`pg_trgm` 虽然通用，但对长句子的相似度计算可能把 stop words 放大。需配置 `zhparser` 或使用 Elasticsearch 的 `icu_analyzer`，但引入额外组件。折中方案：对中文使用 `pg_trgm` 并提高 `similarity_threshold`。

- **向量维度与索引维护成本**  
  1536 维向量在百万级数据下 ANN 索引构建耗时，且写入时需同步更新嵌入。如果 Agent 高频写记忆，需要异步化嵌入生成（用消息队列），否则 LLM 调用会严重阻塞对话流。

- **混合检索参数难调**  
  全文和语义的召回比例、reranker 的阈值、融合方式（RRF、线性加权）对最终效果影响大。我们建议固定一份离线评估集（含 ID 查询、模糊语义查询、混合查询），每改参数必跑一遍。

## 5. 可复用建议

1. **默认采用混合检索，但保留纯后端切换能力**  
   在 OpenClaw 插件的配置里加一个 `retriever_mode: hybrid | fulltext | semantic`，方便 debug 和评估。

2. **实体链接前置**  
   如果 Agent 经常查结构化 ID（工单号、PR ID），先用正则/NER 把这些实体抽出来，直接走精确匹配，余下的文本再走混合搜索。这能极大提高命中率并降低后续推理成本。

3. **监控召回质量**  
   在 memory recall 返回结果时，让 Agent 顺便输出一个 `relevance_score` 或 `useful` 标记（1/0），用来线上反馈微调 reranker 或调整融合权重。

4. **控制记忆量级与时效**  
   对长期记忆做分层：热记忆全量向量索引，冷记忆只保留全文摘要 + 元数据，避免索引膨胀。同时给记忆加时间衰减权重，避免旧信息污染召回。

## 6. 总结

语义搜索不是全文搜索的替代，而是补充。在 Agent 的 memory recall 场景中，硬匹配和模糊理解同样重要。我们的最终方案是：**实体精确匹配 → 全文 + 向量双路召回 → Reranker 精排**。这套组合让 OpenClaw 的 memory 插件在技术文档问答与日常闲聊记忆上都保持稳健，延迟控制在 200 ms 以内。如果你的 Agent 还没做记忆检索，建议先别一步到位上纯向量，从全文开始，按需补语义，再用 reranker 缝合——可控且效果可解释。

---

