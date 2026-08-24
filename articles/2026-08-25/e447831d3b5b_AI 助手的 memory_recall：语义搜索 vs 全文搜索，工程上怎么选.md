---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索，工程上怎么选
feedId: 34603
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 里的 memory_recall 通常不是“存了就完事”，而是 Agent 在对话或执行任务前，根据当前 query 从长期记忆里召回 topN 片段，塞进上下文。召回质量会直接影响后续工具调用和回复准确性。

目前常见两条技术路线：

- **全文搜索**：BM25、倒排索引、FTS5，靠关键词命中。
- **语义搜索**：embedding 向量相似度，靠语义相近。

很多实践者一上来就上向量库，觉得语义搜索更“智能”。但在工程环境里，这个选择并不总是最优。

## 问题

语义搜索擅长模糊表达，比如：

> “上次那个支付回调超时的问题”

它不一定包含“支付”“回调”“超时”这些精确词，但语义向量能兜住。

全文搜索擅长精确 token，比如：

> `ECONNRESET`  
> `/etc/systemd/journald.conf`  
> `ERR_CERT_AUTHORITY_INVALID`

这些字符串如果只靠 embedding，很容易被召回一堆“看起来相关”但实际没用的内容。

所以问题不是“哪个更好”，而是：**memory_recall 场景里，两类查询往往同时存在**。只用语义搜索，精确标识符会漏；只用全文搜索，用户换个说法就查不到。

## 做法 / 步骤

我的做法是混合召回 + 轻量融合，不依赖单一检索器。

### 1. 写入侧：原文与向量同时保存

每条 memory 至少保留：

```text
id, content, tags, created_at, source, embedding
```

写入时生成 embedding，但**不要丢弃原文**。全文搜索需要原文，重排和日志审计也需要原文。

### 2. 召回侧：两路并行

同一个 query 同时跑两路：

```text
full-text: BM25 对 content 检索，top_n = 10
semantic:  query embedding 与 memory embedding 余弦相似度，top_n = 10
```

两路结果可能重叠，也可能互补。

### 3. 融合：用 RRF，不要直接加权

不要直接把 BM25 分数和余弦相似度相加，量纲不同，分数不可比。用 Reciprocal Rank Fusion：

```python
def rrf_fuse(ft_results, sem_results, k=60):
    scores = {}
    for rank, item in enumerate(ft_results):
        scores[item.id] = scores.get(item.id, 0) + 1 / (k + rank + 1)
    for rank, item in enumerate(sem_results):
        scores[item.id] = scores.get(item.id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

融合后取前 20 条候选。

### 4. 重排：可选但有效

如果 memory 量级较大，可以对 20 条候选做一次重排。轻量做法是用交叉编码器打分；更轻量的是直接让 LLM 根据 query 和候选内容判断相关性。最终取 5-8 条注入上下文。

伪代码整体链路：

```python
def recall(query, k=8):
    ft = fulltext_search(query, top_n=10)
    sem = semantic_search(query, top_n=10)
    fused = rrf_fuse(ft, sem)
    candidates = [m for m, _ in fused[:20]]
    ranked = rerank(query, candidates)
    return ranked[:k]
```

## 踩坑点

1. **中文分词破坏精确 token**  
   OpenClaw 的 memory 经常混合中文、英文、代码、日志。默认分词器可能把 `ECONNRESET` 拆坏，导致全文搜索失效。建议对 ASCII token 保留原词，中文单独分词。

2. **语义搜索对短 query 不敏感**  
   比如 query 是 `401`，embedding 可能召回各种认证相关但完全不精确的内容。精确标识符应走全文优先，或给全文路更高权重。

3. **只取语义 topK 会漏掉精确匹配**  
   如果 memory 里有 `redis timeout 5000ms`，语义能召回“Redis 超时”；但如果 query 本身就是 `redis timeout`，全文搜索可能更准。只跑语义路会丢这类 case。

4. **向量更新滞后**  
   memory 内容改了，embedding 没重新生成，导致召回旧版本。写入侧要同步更新向量，不能只改原文。

5. **召回条数过多，污染上下文**  
   低相关记忆塞进 Agent 上下文，反而干扰判断。建议设置相关度阈值，低于阈值不注入。

## 可复用建议

- **默认混合检索**：全文保底，语义扩展。全文路成本低，对代码、命令、编号、路径有效。
- **先过滤再召回**：用 metadata 过滤时间范围、来源插件、任务类型，减少噪声。
- **建立小评测集**：准备 20-50 条真实查询，标注应召回的 memory id，计算 Recall@5、MRR。每次改参数跑一遍，别凭感觉调。
- **记录召回来源**：日志里带上每条记忆的来源和分数，方便定位“为什么没召回”。失败 case 可以反向补 embedding 或加关键词别名。
- **量级小于 1 万条，不必急着上向量数据库**：SQLite FTS5 + numpy 余弦已经够用。工程上简单可控优先。

## 总结

memory_recall 不是“语义搜索替代全文搜索”，而是两者互补。工程上最稳的组合是：**全文保底、语义扩展、RRF 融合、按需重排**。先把召回链路做透明、可评测，再逐步调优，比直接换模型或换库更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/61d48c427819a7f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5d783566bf622130.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1754924df25e1192.png)

