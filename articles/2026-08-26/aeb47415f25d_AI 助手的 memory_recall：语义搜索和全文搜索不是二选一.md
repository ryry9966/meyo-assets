---
title: AI 助手的 memory_recall：语义搜索和全文搜索不是二选一
feedId: 34839
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

memory_recall 在 OpenClaw / Agent / MCP 场景里，经常被简化成“拿 query 去向量库 top_k”。但实际接入插件、工具日志、历史任务后，很多记忆带有精确标识：命令名、报错码、路径、UUID、版本号。对这类内容，语义搜索容易召回“感觉像但不对”的结果；全文搜索又处理不了“上次那个因为超时失败的任务”这种自然语言改写。所以关键不是二选一，而是默认路径怎么搭、什么时候切。

## 问题拆解

- 全文搜索：擅长精确 token、子串、路径、错误码、英文变量名，可解释、可调试，但依赖分词质量，不擅长同义改写。
- 语义搜索：擅长“意图相似但表述不同”，能把模糊表达映射到相关记忆；但专有名词容易被 embedding 平滑，且需要维护 chunk、模型、索引一致性。

典型坏 case：日志里有 `ETIMEDOUT 10.0.0.8:5432`，用户问“数据库连接失败是哪个地址”。只走语义搜索，可能召回“数据库连接失败”相关描述，却漏掉真实错误行；全文用 `ETIMEDOUT` 或 `10.0.0.8` 可以一击命中。

## 做法/步骤

1. 先用 SQLite FTS5 搭可排查的基础检索，不要一上来就上向量库。
   - 建虚拟表：content、source、created_at、metadata_json；
   - 中文检索要验证 tokenizer，必要时接 jieba 或 trigram；
   - 查询时对 title、tag 提权，对 body 降权。

2. 精确通道优先。
   - 把命令名、错误码、路径、UUID、邮箱、版本号抽到 keywords 字段；
   - memory_recall 先做关键词/精确匹配，命中明确标识就直接返回，否则进入模糊通道。

3. 语义通道作为召回扩展，不是唯一来源。
   - Embedding 可选 bge-m3、bge-small-zh 或低延迟 API；
   - 长条目超过 512 token 先切块，块内保留 source 和 position；
   - 向量库可先用 sqlite-vec 跑通，后面再换 pgvector/Qdrant。

4. 混合与重排。
   - 分别取全文 top_n 与语义 top_n，做 RRF：
     `score = 1/(k + rank_fts) + 1/(k + rank_sem)`
   - 语义分低于阈值或精确通道已命中时，回退到全文结果；
   - 不要把 top_k 直接塞给 LLM，给出来源、时间、匹配分，让上层判断是否采用。

5. 小样本评测。
   - 固定 30~50 条查询，标相关记忆 id；
   - 看 recall@5、MRR，记录“该召回没召回”和“召回了不相关”两类 bad case。

## 踩坑点

- FTS5 默认 unicode61 对中文不友好，必须针对中文验证分词。
- 错误码、路径、UUID 被语义搜索稀释，精确通道不可省。
- 长篇日志直接 embedding 会降低相关性，先切块。
- 使用 cosine 时向量要先归一化，否则排序可能偏移。
- memory 过期后向量库仍能召回“幽灵记忆”，需要 hash/updated_at 做增量同步。
- 只看语义相似度分数会被“泛泛相关内容”骗过，要抽查 bad case。

## 可复用建议

- 默认架构：FTS5 全文 + metadata 过滤，语义层可选；不要一步到位上复杂检索栈。
- memory_recall 返回字段建议：`source, score, matched_by(fts/semantic/rrf), timestamp, snippet`。
- 精确关键词先过白名单/黑名单，减少无效嵌入查询。
- 小团队先跑 sqlite-vec + bge-small，QPS 不够再换。
- prompt 里告诉 Agent：记忆结果仅供参考，来源过旧或与当前任务冲突时不要强行引用。

## 总结

语义搜索适合模糊意图，全文搜索适合精确标识。工程上更稳的做法是：全文保底、语义扩展、精确通道优先、RRF 融合、小样本回归。与其争论哪个更好，不如把 memory_recall 做成可解释、可回退、可评估的管道。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/9d0ebec49c5b37e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/9d8e6fa8a66593f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/74dae1273e1fdb82.png)

