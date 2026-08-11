---
title: 混合召回才是正解：在 Agent 记忆里同时用好语义搜索与全文搜索
feedId: 32531
source: 综合讨论
publishedAt: 2026-08-11
---

# 混合召回才是正解：在 Agent 记忆里同时用好语义搜索与全文搜索

## 背景

给 AI 助手接上长期记忆已经成了 Agent 项目的标配需求。无论是通过 MCP 工具把对话摘要写入向量数据库，还是用插件把用户笔记灌进 memory，最后都会遇到同一个问题：**当 Agent 需要从海量记忆中查找相关内容时，到底该用语义搜索（embedding+向量检索），还是传统的全文搜索（BM25/关键词匹配）？**

很多项目一上来就默认“语义搜索更智能”，直接用 OpenAI text-embedding-3-small + pgvector/qdrant 搞定。但在真实业务里，用户记忆往往是**半结构化**、**短文本**、**代码片段**或**精确术语**的混合体。纯靠语义召回，会出现几种典型的翻车场景。这篇帖子记录了我们从踩坑到收敛为一个工程上可复用的混合召回方案的过程，重点面向用 OpenClaw 做 Agent 自动化的同学。

## 问题

一开始，我们给 memory 模块只配了向量检索。用户对话经摘要提炼后存入 PostgreSQL，用 pgvector 做 top_k 召回。很快团队内部工单系统里就出现三类抱怨：

1. **术语漏报**：用户问“上次说的 OIDC 回调地址配在哪个文件？”，向量召回给了一堆关于认证流程的对话，唯独没有那行直接写着 `oidc_callback_uri` 的配置记录。
2. **代码片段命中率极低**：用户说“那个 `react-dom` 版本锁死的 bug 你还记得吗？”，召回结果里塞满了关于 React 优化的讨论，偏偏漏了那句写着 `"react-dom": "18.2.0"` 的笔记。
3. **最近时间窗口偏差**：用户问“下午刚改的 Nginx 超时参数”，向量搜索可能把一周前讨论 Nginx 的对话拉回来，完全无视时间衰减。

核心矛盾在于：**语义理解擅长捕捉“意思相近”，但用户对记忆的回溯往往依赖精确文字匹配；向量模型对短文本、数字、标识符、代码的区分能力远弱于全文索引。**

## 做法

我们没有推倒向量库，而是在召回层做了一层薄薄的“多路召回 + 融合排序”，成本很低。

### 1. 双索引架构

仍以 PostgreSQL 为主存储，每一条记忆记录至少包含：

- `content`：原文
- `embedding`：1536 维向量（text-embedding-3-small）
- `created_at` / `updated_at`
- `user_id` / `session_id` 等租户字段

索引侧同时建立：
- 向量索引（pgvector 的 IVFFlat 或 HNSW）
- 全文索引：对 `content` 列建 GIN 索引，使用 `english` 或对应语言的分词，同时用 `tsvector` 保存原始文本（避免分词丢失精确匹配）

### 2. 双路召回策略

每次 memory recall 查询时，执行两个子查询：

- **语义路**：`ORDER BY embedding <=> query_embedding LIMIT k`（k 取 10~20）
- **全文路**：`WHERE to_tsvector('english', content) @@ plainto_tsquery('english', query) OR content ILIKE '%keyword_token%'`，结合简单正则提取用户问题中的“引号内精确片段”做 ILIKE 硬匹配，返回最多 m 条（m 取 5~10）

注意全文路不是单一 `ts_query`，而是组合了短语匹配和安全的关键词 LIKE（用 `%` 时需对长度和用户输入做限制，防止全表扫描）。

### 3. 融合排序（RRF）

拿到两路结果后，用 **Reciprocal Rank Fusion** 做最终排序，无需训练权重模型。简单实现：

```python
score_map = {}
for rank, item in enumerate(semantic_results):
    score_map[item.id] = score_map.get(item.id, 0) + 1 / (60 + rank)
for rank, item in enumerate(fulltext_results):
    score_map[item.id] = score_map.get(item.id, 0) + 1 / (60 + rank)
final = sorted(score_map, key=score_map.get, reverse=True)[:top_n]
```

`60` 是阻尼常数，工程上能让排在各自列表前几名的结果有更稳定的交叉优势。

### 4. 时间衰减注入

在融合排序前，给每一条记忆计算时间衰减因子，乘入原始权重。我们用了简单的对数衰减：`decay = 1 / (1 + log(1 + days_since_updated))`。对于用户明确提到“最近/刚才/下午”的时间意图，可以将衰减因子调得更激进，或直接限定 `updated_at` 的时间窗。

## 踩坑点

1. **全文索引的分词器与业务语言不匹配**  
   我们用的是 `english` 词典，但用户经常输入中文技术术语。中文场景必须引入 `zhparser` 或切换到支持中文分词的方案（如 jieba + pg_jieba），否则 `to_tsvector('english', ...)` 会把中文每个字当作一个 token，召回效果极差。最简单的逃生通道：保留一个未经分词的 `content` 副本，用 `ILIKE '%term%'` 兜底，但这只是救急，不能长期依赖。

2. **向量搜索的维度诅咒与索引未及时更新**  
   如果使用 IVFFlat 索引，插入大量新记忆后没及时 `REINDEX`，查询计划会退化为全表扫描，延迟从 5ms 飙到 200ms。HNSW 索引写入开销大但查询稳定，需根据写入吞吐选择。我们后来用定时任务每天凌晨重建 IVFFlat 索引，同时监控慢查询。

3. **融合排序后重复结果**  
   同一记忆可能在两路都出现，RRF 本来就会合并，但要注意 ID 的唯一性。如果去重逻辑有 bug，会导致返回列表出现重复条目，Agent 容易在重复上下文上反复引用，显得很啰嗦。

4. **成本陷阱**  
   每路召回都调用 embedding API 成本不高，但每次用户请求都会触发一个额外的 embedding 查询。在 QPS 变高后，需要把用户查询的 embedding 做短期缓存（我们的缓存 key 是 `query_text + user_id`，TTL 5 分钟），简单有效。

## 可复用建议

- **不要为了追求“智能”而丢掉精确搜索**。即使是 GPT-4 作为 Agent，也需要在 memory 里快速捞出代码片段、配置项、ID。全文搜索的性价比极高。
- **双路召回 + RRF 是一个低心智负担的起点**。不需要预先判断用户意图是语义还是关键词，让结果自然竞争。
- **时间信号比预料中更重要**。Agent 需要“近期记忆”的场景远多于需要古老上下文，把 `updated_at` 做成可调的滤镜，而非仅靠排序。
- **监控每路召回的命中率**。我们在 metadata 里记录了每条记忆被哪路召回命中，定期统计可以发现索引退化或分词问题。
- **别让 Agent 自己决定召回策略**。把 memory recall 做成确定性工具（MCP function），不要给 Agent 暴露 `search_type` 参数让他选，避免产生不稳定行为。

## 总结

我们没有发明任何新算法，只是把信息检索领域几十年的老套路——多路召回、RRF 融合、时间衰减——挪到了 Agent 的记忆层。结果就是工单里关于记忆不准的抱怨减少了 80% 以上。如果你正在用 OpenClaw 做 Agent 的 memory 插件，我建议从一开始就留好全文索引的路，哪怕第一步不启用，也预留 `content_tsv` 列和对应的索引字段。未来当你需要 Agent 记住某个精确的接口参数时，你会感谢这个决定。

---

