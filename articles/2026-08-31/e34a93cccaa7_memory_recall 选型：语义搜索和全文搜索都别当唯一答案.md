---
title: memory_recall 选型：语义搜索和全文搜索都别当唯一答案
feedId: 35470
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

给 Agent 接 memory_recall 时，常见两种实现：全文搜索（SQLite FTS5 / BM25）和语义搜索（embedding + 向量相似度）。很多人一开始就上向量库，感觉更“智能”，但在 OpenClaw / Agent / MCP 这类自动化链路里，memory_recall 的要求不是“聊起来顺”，而是“在正确时间把正确上下文捞回来”。

一个典型场景是：助手需要从过往会话、操作记录、插件输出中召回相关 memory，再喂给 LLM 做下一步决策。这时召回的可复现、延迟和排障成本，往往比纯粹的相关性更重要。

## 问题

什么时候该用语义搜索，什么时候该用全文搜索？

- 语义搜索适合：短句、改写、跨语言、用户描述模糊。
- 全文搜索适合：精确 ID、路径、报错码、函数名、日志片段、中文专有名词。

直接二选一容易翻车。例如用户问“上次那个数据库连接超时的处理”，语义搜索能理解“超时”和“timeout”；但用户问“/workspace/app/config.yaml 里的 redis timeout”，全文搜索能直接定位，语义搜索可能因为 chunk 切得太碎或 embedding 泛化而召回一堆不相关配置。

## 做法/步骤

1. 先建全文索引，优先用 SQLite FTS5。中文环境建议 `tokenize='trigram'`，避免默认 unicode61 对中文整句不友好：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  tokenize='trigram'
);
```

2. 语义搜索作为第二路，只在需要时启用。可以用本地 embedding 或远程 API，但要把 chunk 控制在 200-600 字/块，避免过短丢失上下文、过长稀释语义。

3. 统一封装 memory_recall 接口，不要到处直接查库：

```python
def memory_recall(query, limit=10, semantic=False):
    hits = fts_search(query, limit)
    if semantic and len(hits) < limit:
        vec_hits = semantic_search(query, limit)
        hits = rrf_merge(hits, vec_hits)
    return hits[:limit]
```

4. 元数据过滤先于检索：source、agent、time_range 先过滤，再在候选集里做 FTS / 向量。

5. 建立小型回归集，记录 recall@5，不要凭感觉调参。

## 踩坑点

- 中文 FTS 用默认 tokenizer 会把整句当一个 token，导致召回差；trigram 能解决子串匹配，但会有噪音，需要限制结果数。
- 语义搜索对代码、路径、版本号等精确 token 不敏感，embedding 相似度高不代表可执行。
- 向量库返回结果难以排障：为什么召回这条？需要保留原始文本和相似度分数，最好记录 recall_source。
- 全文搜索对拼写错误、同义词、跨语言很弱，这时候才需要 semantic 补位。
- 不要把 chat 日志和 memory 混在一个库，污染严重以后召回质量断崖式下降。memory 应该是经过筛选、压缩、标注的条目。

## 可复用建议

- 默认全文，语义可选。先跑通 FTS5，再按失败 case 决定是否上 embedding。
- memory_recall 的返回必须带 source、score、timestamp，方便 Agent 判断可信度，也方便人肉排障。
- 对精确短语保留 exact match 优先：命中路径 / ID 时直接返回，不参与语义重排。
- 混合召回用 RRF 比按分数线性相加更稳，避免两个分数分布不一致。
- 每周看一次低召回 query，把需要语义化的表达沉淀成同义词或规则，不要只靠 embedding 自己“悟”。

## 总结

语义搜索和全文搜索不是谁替代谁。全文负责“找得到”，语义负责“想得到”。在 OpenClaw 这类工程链路里，先用 FTS5 把可复现性和低延迟拿到手，再用语义搜索补模糊意图，最后通过统一 recall 接口和回归集控制质量。memory_recall 如果做成黑盒，后面排障会非常痛苦；让它返回来源和分数，比换哪个向量库更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ad269be1b087d99f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/1422af590b8d8388.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/87a1fbe15c70ad7b.png)

