---
title: AI 助手的 memory_recall：语义搜索和全文搜索，别二选一
feedId: 34745
source: 综合讨论
publishedAt: 2026-08-26
---

# 记忆召回：语义搜索与全文搜索，先别急着二选一

## 背景

在 OpenClaw 这类 agent 工程里，`memory_recall` 通常不是简单的“读档”，而是在任务执行前、中、后从记忆库召回相关上下文。记忆来源可能包括工具输出、用户偏好、历史决策、报错记录、代码片段。召回质量直接决定模型是否会产生幻觉，或者重复踩同一个坑。

很多实现一开始会用全文搜索（FTS/BM25），因为它容易落地；后来听说语义搜索更“聪明”，就全面迁移到 embedding。结果往往不是更好，而是出现新的静默失败。

## 问题：两种搜索解决的是不同问题

全文搜索适合精确信息：命令、路径、报错码、函数名、版本号、配置项。它的局限是词形不匹配就搜不到，例如“磁盘写满”和 `no space left on device`。

语义搜索适合模糊意图、同义改写、跨段落归纳。它的局限是：

- 对短查询、代码符号、日志 token 不稳定；
- embedding 有模型依赖，换模型后旧向量不可比；
- 会召回“感觉相关但实际无用”的内容，且不擅长处理必须精确排除的字段。

在 `memory_recall` 场景里，两者不是替代关系，而是输入类型差异。精确标识符不能只靠语义，叙述性记忆不能只靠字面命中。

## 做法：先结构化过滤，再混合召回

推荐的最小路径是：**结构化过滤 -> FTS + 向量并行召回 -> 归一化排序 -> 可选重排**。

结构上可以只用 SQLite：

- 用 FTS5 建全文索引，存储原始文本；
- 用 `sqlite-vec` 或外部向量库存 embedding；
- memory 表至少包含：`id`、`type`、`content`、`metadata`、`created_at`、`embedding_model`、`embedding_version`。

查询时：

1. 先按 `agent_id / session_id / type / 时间窗` 过滤，缩小候选；
2. FTS 查询取 top_k；
3. 向量查询取 top_k；
4. 用 RRF 融合：`score = sum(1 / (k + rank))`，建议 k 取 60；
5. 若对准确率要求高，再加 cross-encoder 对 top 10-20 重排。

伪代码：

```text
docs = structured_filter(agent_id, type, time_window)
fts_hits = fts_search(query, docs, top_k=10)
vec_hits = vector_search(embed(query), docs, top_k=10)
merged = rrf(fts_hits, vec_hits, k=60)
return rerank(query, merged, top_n=8)
```

这一套不复杂，但能把“精确召回”和“模糊召回”拆成两条明确通道，避免相互污染。

## 踩坑点

1. **全量迁移到向量搜索后，命令和报错码召回变差**。精确 token 在 embedding 空间里不一定相邻。保留 FTS 通道，对 `error_code`、`command`、`path` 这类字段不要只依赖 embedding。
2. **SQLite FTS5 默认分词对中文不友好**。`unicode61` 会按字切分，中文短语召回容易失控。建议使用 trigram 分词，或先做外部中文分词再写入 FTS。
3. **不同 embedding 模型不可混用**。旧向量和新向量比较会产生系统性偏差。写入时记录 `embedding_model` 和 `dim`，换模型必须重建索引。
4. **RRF 分数不能直接相加**。FTS 分数和余弦相似度量纲不同，直接相加会被某一通道主导。RRF 或 min-max 归一化更稳。
5. **冷启动阶段不要过度设计**。记忆只有几百条时，FTS 足够。向量搜索收益有限，先解决结构化过滤和内容写入质量。

## 可复用建议

- **先建 eval 集再改召回**。从真实任务中抽 20-50 个查询，标注相关文档，用 `hit@5 / MRR` 评估。没有指标时，任何“语义搜索更好”都只是感觉。
- **给 memory 分层**：标题、命令、ID、报错码走 FTS；总结、决策理由、用户偏好走语义。
- **写入时保留原始文本**：向量索引只是补充，不要只存摘要。FTS 需要完整原文命中。
- **记录 model/version**：每次 embedding 请求附上模型名，便于将来迁移。
- **关注静默失败**：向量搜索低分返回、FTS 零结果但向量有结果，这类 case 要日志化，积累成样本。

## 总结

语义搜索不是全文搜索的升级版，它解决的是“意思相近”，全文搜索解决的是“字符精确”。在 agent `memory_recall` 中，最佳实践通常是先用结构化过滤控制候选范围，再用 FTS 保证精确召回，用向量处理语义模糊，最后用 RRF 融合、必要时重排。先上全文搜索，配备评估集，再逐步引入语义通道，比一步到位更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/f25d361c24cb1465.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/f7e587610521618d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/44a8d127bef3e05f.png)

