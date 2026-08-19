---
title: AI 助手 memory_recall：语义搜索不是银弹，先做好全文检索再上向量
feedId: 33822
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

给 OpenClaw 这类 Agent 加长期记忆时，`memory_recall` 是绕不开的模块。无论你是用插件、MCP server 还是直接在 memory pipeline 里写召回逻辑，最终都会面临同一个选择：**用全文搜索还是语义搜索？**

社区里常见两种声音：一种认为 FTS/BM25 足够快、可解释、维护成本低；另一种认为语义搜索更“智能”，能理解改写和意图。实际工程里，单独押注哪一边都容易踩坑。这篇文章记录我在 OpenClaw 环境下的实践结论：**语义搜索不是替代品，而是补充；先做好全文检索和元数据过滤，再按需加向量召回。**

## 问题

全文搜索的典型问题：

- 用户说“我之前提过怎么调超时”，但记忆里存的是“timeout 设置”，关键词搜不到。
- 中文分词不当，“Agent 上下文太长”和“上下文窗口太大”无法命中。

语义搜索的典型问题：

- 向量召回“看起来相关”的碎片很多，精确术语反而被淹没。
- 每次召回要调 embedding，延迟和成本上升；本地模型质量不稳定，API 模型有网络和隐私问题。
- 编辑/删除记忆后，向量库不同步，召回脏数据。

所以真正的命题不是“哪个好”，而是：**在一个 memory_recall 流程里，如何用最小成本拿到可用的候选集，再交给 LLM 做最终判断。**

## 做法与步骤

我的基线实现很简单：**SQLite FTS5 全文索引 + 元数据过滤**。先不接向量库。

建表结构大致如下：

```sql
CREATE TABLE memory_entries (
  id TEXT PRIMARY KEY,
  content TEXT NOT NULL,
  memory_type TEXT NOT NULL DEFAULT 'note',
  agent_id TEXT,
  created_at INTEGER,
  updated_at INTEGER
);

CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  memory_type,
  content='memory_entries',
  content_rowid='rowid',
  tokenize='trigram'
);
```

`tokenize='trigram'` 是为了缓解中文分词问题。如果你的内容以英文为主，可以换成 `unicode61`，但中文场景 trigram 更稳。

`memory_recall(query, limit)` 先跑 FTS5：

```sql
SELECT id, content, memory_type, created_at
FROM memory_fts
WHERE memory_fts MATCH ?
  AND agent_id = ?
ORDER BY rank
LIMIT ?;
```

这一步已经能解决大量精确术语、关键词召回。只有当真实场景里出现“同义改写找不到”时，再加语义搜索。我的语义层用 `sqlite-vec` 存 embedding，避免引入额外向量数据库。

最终 recall 流程：

1. 先做**权限/范围过滤**：agent_id、memory_type、时间窗口。
2. 并行跑 FTS5 和语义搜索，各自取 Top 20~30。
3. 用 **RRF（Reciprocal Rank Fusion）** 融合两路结果，而不是直接加权分数。向量相似度和 BM25 分数量纲不同，加权容易偏。
4. 按 memory_type 做时间衰减：偏好类记忆不能只看新；短时事实类可以优先新数据。
5. 返回时保留 `source` 和 `score`，让后续 LLM 判断这条记忆是否可信。

伪代码：

```python
def memory_recall(query, agent_id, limit=20):
    fts_candidates = fts_search(query, agent_id, top_k=30)
    vec_candidates = vector_search(query, agent_id, top_k=30)
    fused = rrf_fuse(fts_candidates, vec_candidates)
    filtered = apply_time_decay(fused, memory_type_weight)
    return filtered[:limit]
```

## 踩坑点

**1. SQLite FTS5 中文分词**
默认 tokenizer 对中文整句切词，导致“超时设置”和“设置超时”不能互相命中。`trigram` 能解决大部分，但会增大索引体积。如果记忆量大，可以先用 jieba 等中文分词后写入一个 `tokens` 列，再对该列建 FTS。

**2. 语义搜索 Top K 不能太小**
向量相似度召回容易漏掉精确术语，所以 top_k 建议 20~50，再交给 RRF 融合。如果 top_k 太小，语义搜索的优势发挥不出来；太大会引入大量噪声，拖慢后续 LLM 处理。

**3. 向量库不同步**
memory 有编辑和删除时，一定要同步更新向量表，否则会召回已删除内容。建议每次写入/更新时在同一事务里更新 embedding，或者定义定期重建向量的任务。脏数据比漏召回更影响体验。

**4. 时间衰减不能全局套用**
如果所有记忆都按“越新越重要”衰减，会导致“用户长期偏好”被短期噪音覆盖。建议按 memory_type 区分：偏好/原则类衰减慢，事实/操作类可以快。

**5. 不要把 recall 结果直接当答案**
memory_recall 返回的是候选，不是最终答案。最好让 LLM 根据召回内容判断是否采用，或者做一次小范围重排。否则召回里混入一条相似但上下文冲突的记忆，会直接带偏 Agent 行为。

## 可复用建议

- **先上 FTS5，别急着上向量库。** 如果你的记忆量小于 5 万条，SQLite FTS5 + 元数据过滤足够覆盖大多数精确召回场景。
- **语义搜索按需引入**，只在出现真实同义改写问题时启用。本地 embedding 可以先用 `bge-small-zh` 或类似轻量模型，避免 API 延迟。
- **混合检索用 RRF，不要用加权分数。** 这是工程上最省心的融合策略。
- **建立固定 query 集做回归。** 比如准备 30 条 query，包含精确术语、同义改写、跨会话引用，每次改动召回逻辑后跑一遍命中率，比凭感觉调参可靠。
- **小规模不要上 Elasticsearch/PGVector。** SQLite FTS5 + sqlite-vec 在本地足够轻量，也容易跟着 OpenClaw 一起分发和备份。

## 总结

`memory_recall` 的目标不是“找到最相似的向量”，而是**用最小噪声把可能有用的上下文递交给 LLM**。全文搜索负责精确、可解释、低成本召回；语义搜索负责覆盖改写和模糊意图。先做好全文索引和元数据过滤，再用向量做补充，比一开始就上复杂语义架构更务实。

对于 OpenClaw 这类 Agent 记忆模块，工程上的优先级应该是：**过滤 > 全文 > 混合 > 重排 > 语义精度调优**。语义搜索值得上，但不值得第一步就上。

---

