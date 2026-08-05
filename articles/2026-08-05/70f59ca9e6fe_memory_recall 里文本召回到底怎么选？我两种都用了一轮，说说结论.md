---
title: memory_recall 里文本召回到底怎么选？我两种都用了一轮，说说结论
feedId: 31701
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

把 OpenClaw 接入日常自动化之后，最烦的不是 Agent 不会干活，而是它“记性差”。为了让它记住关键上下文，我给插件体系加了 memory_recall 能力。最开始图省事，直接全表 LIKE 查询，发现召回质量很不稳定。后来换了向量语义搜索，觉得“高级了”，结果一堆新坑。两边都跑了一段时间，把结论整理一下，给后面接 Agent 存储的朋友做参考。

## 问题：两类搜索根本不是一回事

全文搜索（比如 SQLite FTS5、pg_trgm）解决的是“字面对得上”的问题。适合查配置项、错误码、变量名，因为这类记忆里包含符号和精确术语。语义搜索（embedding + 向量库）解决的是“意思差不多”的问题。适合查用户偏好、历史决策、模糊意图，因为这类内容写法多样，用户不会按固定格式说话。

问题在于，OpenClaw 的插件系统里，memory_recall 往往是统一入口，用户不会告诉存储层这次查询属于哪种类型。写死一种搜索方式，就会在另一半场景里失效。

## 做法：双通道召回 + 轻量重排

我现在的做法是在插件层把两步串起来。

第一步，双通道并行。对同一条 query，同时走 FTS（BM25 打分）和向量检索（余弦相似度），各取 Top 30。这两个通道可以并行，延迟没有叠加。

第二步，轻量重排。用 Python 写一个极简 RRF（Reciprocal Rank Fusion）：

```python
def rrf(rank1, rank2, k=60):
    scores = {}
    for rank_list in (rank1, rank2):
        for pos, doc_id in enumerate(rank_list):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + pos + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

不需要精排模型，两个列表各取前 30 合并后取 Top 10，就足够应付大多数场景。

第三步，给召回结果补过滤条件。在 plugins 配置里加一个 `memory_recall.mode` 字段：

- `auto`（默认）：双通道 + RRF
- `keyword`：只走 FTS，适合查代码片段、日志
- `semantic`：只走向量，适合查用户偏好类的内容

这一步很重要，它给了上层 Agent 一个按需选择的权利，而不是替 Agent 做决定。OpenClaw 插件机制里支持在 args 里透传这个字段，配置在 `~/.openclaw/memory.yaml` 的 `recall_mode` 键下。

## 踩坑点

1. **向量库召回假阳性高**。embedding 对符号和数字不敏感。查 `"OPENAI_API_KEY"` 这种词，向量召回能把 `"ANTHROPIC_API_KEY"` 也捞出来，看起来相似但实际不相关。
2. **全文搜索容忍不了“人话”**。用户上次说“部署时总超时”，这次搜“环境卡住了”，FTS 一个字匹配不上。这类表述差异必须靠语义维度。
3. **别为召回问题引入重排序模型**。OpenClaw 插件体系里很多用户跑在低配 NAS 或 4G 内存的云主机上，bge-reranker 这类模型加载就吃掉 500MB+ 内存。RRF 足够用。
4. **向量检索必须配 metadata 过滤**。如果存储层是全量记忆，不管时间范围直接搜，最新记忆会被淹没，因为没有时间衰减。必须在召回层限制 `last_accessed_at` 范围。

## 可复用建议

如果你的场景以精确匹配为主（代码片段、命令、配置），只做 FTS 就行，别盲目上向量。

如果必须处理模糊语义，双通道是最稳的方案。落库时同时生成 embedding 和打包 FTS 索引，没有额外负担。

如果每条记忆都会长期累积，建议按周做一次 embedding 刷新，因为 embedding 模型升级后旧的向量和新向量维度一致但语义空间有偏移，混合使用会劣化。定期重刷，成本可控。

## 总结

语义搜索和全文搜索不存在谁替代谁。做 memory_recall 时，用双通道融合是成本最低、效果最稳的选择。单通道在特定场景下更简单，但会给上层 Agent 带来不可预知的召回缺陷。与其纠结选哪种，不如先承认两者互补，然后用 10 行代码做一个 RRF 融合入口，剩下的交给参数调优。

---

