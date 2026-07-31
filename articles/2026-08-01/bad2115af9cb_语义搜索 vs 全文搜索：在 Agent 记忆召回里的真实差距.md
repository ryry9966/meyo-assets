---
title: 语义搜索 vs 全文搜索：在 Agent 记忆召回里的真实差距
feedId: 31124
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：记忆召回为什么会变成一个选择题

在 OpenClaw 或 MCP Agent 这类需要长期运行的自动化助手当中，memory recall 不是锦上添花，而是基础设施。要让 Agent 记住上下文、用户偏好、历史决策，就必须对“过去说过/做过的事”进行高效检索。

目前最主流的两条技术路径是：

- **全文搜索（Full-text search）**：基于关键词匹配，常见的如 BM25、PostgreSQL 的 `tsvector`、Elasticsearch。
- **语义搜索（Semantic search）**：基于句子/段落的嵌入向量（embedding），用余弦相似度检索语义相近的内容。

RAG 热潮让很多人直接上了语义搜索，但真实工程里的体验往往是：语义搜索能带来惊喜，也能带来灾难；全文搜索看起来笨重，但在很多场景下反而更可控。

这篇文章不会给你一个“哪个更好”的绝对答案，而是基于在 OpenClaw/MCP 插件环境下踩过的坑，给出一个务实的选型和混用思路。

## 问题：语义搜索并不总是更聪明

以客服场景为例。我们在一个 MCP 插件里实现了对话历史召回，最初全用语义搜索（OpenAI text-embedding-3-small + pgvector）。结果出现两类典型问题：

1. **“为什么召回这段？”**：用户问“订单取消后多久退款”，语义搜索把几个月前一条“我不想续费了”的感慨排在了最前面，因为语义上确实接近，但对解决问题毫无帮助。
2. **精确匹配完全失效**：用户输入一个订单号，想查上次沟通中是否提到过，语义搜索因为该订单号只在文本中出现过一次、且周围语义平淡，直接把它丢到后面，而全文搜索可以 100% 命中。

反过来，全文搜索在需要模糊关联时显得无力。用户说“上次那个支付失败的问题”，全文搜索只匹配“支付失败”关键词，但历史记录里当时用的是“银行扣款异常”，就没能召回。

所以核心矛盾不是技术高下，而是：**你要引擎精确理解问题，还是精确匹配字面；在何种场景下，你更承受不起哪一种失败。**

## 做法与步骤：在 MCP 内搭一套混合召回

在 OpenClaw 生态里，我们可以通过 MCP server 暴露 memory recall 工具。这里给出一个轻量级方案，不依赖 Elasticsearch 这类重型组件，用 PostgreSQL + pgvector 同时实现全文和语义检索，最后做结果融合。

### 1. 数据结构

```sql
CREATE TABLE memory_chunks (
    id uuid PRIMARY KEY,
    session_id text,
    content text,
    embedding vector(768),          -- 语义向量，维度取决于模型
    fts_vector tsvector,            -- 分词向量，用于全文检索
    created_at timestamptz
);
```

`fts_vector` 通过触发器或应用层每次写入时由 `to_tsvector('simple', content)` 生成。这里用 `'simple'` 避免分词器对数字、订单号等产生过度切分。

### 2. 双路检索

- **全文检索**：使用 `ts_rank` 或 `ts_rank_cd` 排序，查询时用 `plainto_tsquery` 或 `phraseto_tsquery`。
- **语义检索**：查询前先把用户问题转换成 embedding，再用余弦距离排序。

```sql
-- 全文
SELECT id, content, ts_rank(fts_vector, query) AS rank
FROM memory_chunks, plainto_tsquery('simple', '订单取消后多久退款') query
WHERE fts_vector @@ query
ORDER BY rank DESC LIMIT 20;

-- 语义
SELECT id, content, 1 - (embedding <=> query_embedding::vector) AS similarity
FROM memory_chunks
ORDER BY embedding <=> query_embedding::vector LIMIT 20;
```

### 3. 融合排序：RRF (Reciprocal Rank Fusion)

两路各自产出 Top-K 结果后，用 RRF 合并，无需调权重，工程上足够稳定。

```
score(doc) = 1/(60 + rank_fts) + 1/(60 + rank_semantic)
```

在 MCP 工具函数里拿到最终融合后的 top-N，返回给 Agent，同时附上来源类型（`fts`/`semantic`），方便后续诊断。

## 踩坑点

1. **Embedding 模型选择与多语言**
   如果内容里中英文混杂，用仅训练于英文的 embedding 模型会严重退化。`text-embedding-3-small` 多语言表现尚可，但长文本截断会导致信息丢失。建议对长文本先用规则切段，每段不超过 512 tokens。

2. **全文索引的分词陷阱**
   订单号、序列号、邮箱这种字符串，在默认分词器下可能被切成多个 token，导致无法精确匹配。我们最终使用 `simple` 词典，同时对这类实体单独建了一个 `text` 列做 `pg_trgm` 模糊匹配作为补充。

3. **混合检索的性能开销**
   双路检索 + 融合排序会在高并发下造成明显延迟。解决办法：
   - 对 embedding 做缓存：相同 query 在短期内复用。
   - 全文检索部分使用物化视图或定期刷新的 `tsvector`，避免实时计算。
   - 设置召回窗口上限（比如只检索最近 30 天的记忆），通过 `created_at` 分区。

4. **排序结果难以解释**
   给到 Agent 的上下文中，一定要带上来路标记和分数。一旦 Agent 输出错误，你可以快速定位是全文没命中还是语义发散了，而不是盲目调整。

## 可复用建议

- **先上全文，再补语义**
  如果你的记忆内容以事实、实体、数字为主（日志、订单、诊断信息），直接上全文搜索，性价比最高。语义搜索追加作为“模糊召回补充”，而不是主力。

- **用监控决定权重**
  在 MCP 工具里埋点，统计两路召回的命中率、无意义召回率。如果语义路连续给出低质量结果，就在那类 session 上动态禁用语义，只用全文。

- **别忘时间衰减**
  很多场景下，近期记忆比远期记忆更有用。可以在 RRF 之外再乘上一个指数衰减因子，避免过期信息排到第一位。

- **MCP 工具设计：暴露 debug 参数**
  在 MCP memory_recall 工具里增加 `debug=true` 时，返回每条结果的来源、分数、距离。这样当 Agent 行为怪异，你可以在 OpenClaw 的前端或日志中快速回溯。

## 总结

语义搜索与全文搜索在 memory recall 中不是取代关系，而是互补关系。核心不在于技术栈多先进，而在于你是否清楚：你的记忆内容长什么样，你的失败更倾向于“丢结果”还是“多噪声”。

在 OpenClaw/MCP 这类可组合架构下，保持召回路径可观测、可降级，远比无脑上向量数据库重要。先用简单规则和全文搜索把骨架搭起来，语义搜索作为锦上添花的第二路，跑一个月数据之后再看真正需要优化什么——这种做法，工程上远比一上来就追求“全语义”要健康。

---

