---
title: OpenClaw 实践：memory_recall 选语义还是全文？混合召回才是解药
feedId: 33283
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 里跑 Agent 时间一长，memory 表会积累大量历史：工具输出、报错栈、用户偏好、临时决策。`memory_recall` 是这些记忆的召回入口，直接影响后续推理质量。最近我把一个跑日志排障的 Agent 从纯语义搜索切到混合检索，踩了几个坑，记录一下。

OpenClaw 的 memory_recall 支持三种模式：`semantic`（向量相似度）、`fulltext`（BM25/FTS5）、`hybrid`（混合检索 + RRF 融合）。默认配置往往只开 semantic，但实际任务里单靠一种召回经常翻车。

## 问题：语义不是银弹，全文也不是废物

语义搜索擅长同义改写。比如问“昨天数据库连不上是什么原因”，它能召回一条写有“postgres connection reset by peer”的记忆。但它的缺陷也很明显：

- 对精确 token 不敏感，错误码、UUID、路径、命令名容易被近义词干扰。
- 排序不可解释，有时相似度 0.7 的记忆反而不如 0.5 的相关。
- 依赖 embedding 模型质量，中英文混输场景尤其明显。

全文搜索正好相反：

- 对 `ECONNREFUSED 127.0.0.1:5432` 这类精确匹配，BM25 能直接命中。
- 速度快，索引占用小，结果可解释。
- 但遇到同义改写、口语化表达，召回率会断崖式下降。

我在同一批 2k 条 memory 上做了简单对比：15 条 query，semantic 模式 recall@5 是 0.73，fulltext 只有 0.47。但拆开看，fulltext 在错误码、文件名、工具名上的命中率是 100%，而 semantic 在这些 case 上反而会混入无关的近义词条目。

## 做法 / 步骤

1. **固定数据集**：把 memory 表导出成 JSONL，保留 `content`、`created_at`、`metadata`，不删除旧数据。
2. **先跑 semantic only**：OpenClaw 配置 `search_type: semantic`，embedding 用 `bge-m3`，`top_k=8`。记录每条 query 的召回结果和人工相关判断。
3. **再跑 fulltext only**：切到 `fulltext`，确认底层 FTS5 索引已建立。中文场景要检查 tokenizer，默认 unicode61 对中文约等于废。
4. **最后跑 hybrid**：配置 `search_type: hybrid`，开启 `rrf` 融合，`rrf_k=60`。把 `top_k` 设为 10，观察融合后的排序。
5. **观察结果**：精确错误码 query 在 hybrid 下排名稳定第一；语义类 query 也能召回 fulltext 漏掉的改写条目。总体 recall@5 提升到 0.87。

部分配置示意：

```yaml
memory_recall:
  search_type: hybrid
  top_k: 10
  hybrid:
    semantic_weight: 0.6
    fulltext_weight: 0.4
    rrf_k: 60
  fulltext:
    tokenizer: simple
```

## 踩坑点

- **中文分词**：FTS5 默认 tokenizer 对中文不友好，需要改为 `simple` 或外挂 jieba 等分词。否则全文检索中文等于摆设。
- **RRF 参数**：`rrf_k=60` 是常用值，但任务不同需要调。k 太大则精确匹配权重下降，太小则语义结果过强。
- **阈值校准**：不要直接信任向量相似度。同一个 embedding 模型下，0.55 可能已经相关，也可能完全无关。建议用标注集校准。
- **存储成本**：本地 SQLite 存向量会拖慢检索，尤其 memory 表超过 1 万条时。考虑外接 qdrant 或 milvus。
- **时效性**：混合检索不解决过期记忆问题。如果 memory 没有时间衰减或元数据过滤，recall 可能把三个月前的旧配置排到前面。

## 可复用建议

- **不要二选一**。工程上默认 hybrid，保留 semantic / fulltext 作为 fallback。
- **按 query 路由**：query 含错误码、路径、命令名、UUID 时，优先 fulltext 或提高 BM25 权重；自然语言问句走 semantic 为主。
- **返回 score 并让模型感知**：在 prompt 里要求对低分记忆标注不确定性，减少错误引用。
- **建立回归评测集**：至少 30 条真实 query，固定记录两种模式的 recall@5。每次换 embedding 或调阈值后重跑。
- **中文场景先解决 tokenizer**，否则 fulltext 在 hybrid 里只会拖后腿。

## 总结

语义搜索解决“意思相近”，全文搜索解决“字符精确”。在 OpenClaw 的 memory_recall 里，两者不是替代关系，而是互补。与其纠结哪个更好，不如先上 hybrid，再用 query 路由和阈值校准把各自优势发挥出来。最后保持可观测性，别让记忆召回成为 Agent 的隐性瓶颈。

---

