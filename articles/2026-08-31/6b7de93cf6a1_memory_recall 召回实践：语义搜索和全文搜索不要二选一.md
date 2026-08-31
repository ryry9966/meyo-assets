---
title: memory_recall 召回实践：语义搜索和全文搜索不要二选一
feedId: 35577
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw / Agent / MCP 插件这类项目里，memory_recall 通常不是“存得越多越好”，而是“需要时能把相关记忆捞回来”。召回层常见两条路线：全文搜索（FTS5 / BM25）和语义搜索（embedding + 向量相似度）。两者的差异不是效果好坏，而是失败模式完全不同。

## 问题

全文搜索擅长精确 token：报错码、版本号、函数名、路径、ID。缺点是用户换一种说法就召回不到，比如“超时重试的配置”和“timeout retry”需要同义词或词典。

语义搜索擅长意图和改写：把“登录卡住”对上“认证超时”。缺点是它对短文本、代码片段、编号不敏感，甚至会把 v1.4.2 和 v1.4.3 当成高相似，因为它看的是上下文相似而不是 token 精确。

如果你的 memory_recall 只接一路，运维时一定会遇到另一个方向的漏召回。

## 做法/步骤

1. 先定义召回单元。不要直接索引原始日志。建议按“会话摘要、决策记录、工具返回、用户偏好”切成小块，并带上 `session_id / user_id / created_at / scope` 元数据。
2. 先上全文索引。哪怕是 SQLite FTS5 也比没有强。建表后可以按 BM25 取值。
3. 再补语义索引。把同一批 chunk 送到本地 embedding 模型，存进 SQLite-vec、pgvector 或现有向量库均可。
4. 查询时两路各自取 top_k=20，然后用 RRF 融合，而不是直接相加分数。
5. 过滤和重排放在融合之后，按时间范围、会话、工具域先滤掉明显无关项。

融合可以用极短代码完成：

```python
def rrf(ranks_a, ranks_b, k=60):
    scores = {}
    for rank, doc_id in enumerate(ranks_a, start=1):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    for rank, doc_id in enumerate(ranks_b, start=1):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    return sorted(scores.items(), key=lambda x: -x[1])
```

RRF 的好处是不关心余弦值和 BM25 分数的量纲，只关心相对名次，适合两路异构召回。

## 踩坑点

- **不要把向量相似度当阈值硬卡**。不同查询、不同 chunk 长度的相似度分布会漂，0.75 可能今天漏、明天多。
- **不要只用原始日志做语义 chunk**。日志噪声大，嵌入后相似度会向高频词靠拢，召回很多“看起来像”但无关的片段。
- **避免短 chunk 直接向量化**。错误码、版本号、路径这类内容，优先走 FTS；非要进向量库，也应与上下文一起编码。
- **嵌入模型更换必须重建索引**。混用不同模型的向量做相似度比较，排序基本失效。
- **不要忽略过滤条件**。跨会话召回是 memory 污染的主要来源，先按 user/session/agent scope 收窄。

## 可复用建议

如果今天要落地：

- 场景偏“精确查找”：FTS5 足够，不要为了语义而语义。
- 场景偏“模糊回忆”：语义搜索优先，但保留 FTS 作为兜底。
- 大多数 agent memory 是混合场景，推荐用 FTS + vector + RRF，而不是二选一。
- 先量化失败案例：统计漏召回是由于改写、同义词、还是编号/ID，再决定补哪一路。

## 总结

语义搜索和全文搜索不是替代关系。全文搜索保证你找得到说过的东西，语义搜索保证你找得到意思相近的东西。对 memory_recall 来说，工程上更稳的做法是：FTS 先兜底，语义后补充，最后用 RRF 融合。不要一上来就把所有希望压在向量相似度上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8f1da9fb423a3dad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a77f1a09e0db0577.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/66df0d0a5d2fe9e4.png)

