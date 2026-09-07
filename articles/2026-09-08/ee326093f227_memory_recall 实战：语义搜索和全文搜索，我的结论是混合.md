---
title: memory_recall 实战：语义搜索和全文搜索，我的结论是混合
feedId: 36530
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

Agent 的 `memory_recall` 是个很朴素的工具：把历史对话和笔记存下来，需要时检索几条塞回上下文。在 OpenClaw 插件生态里，常见两种实现：向量库做语义搜索，或 SQLite FTS 做全文搜索。社区里经常争论哪个更好。我前阵子给自己搭的助手重写了记忆检索，两条路都完整踩了一遍，结论先给出：单用哪个都有明显盲区，混合才是默认答案。

## 问题：各自的盲区

语义搜索擅长同义改写——“怎么处理超时”能召回“请求长时间未响应后连接断开”这类记录。但它对精确 token 很不敏感：错误码 `E_CONN_104`、函数名 `process_batch_v2`、某个配置路径，在 embedding 空间里会被语义“长得像”的条目淹没。

全文搜索反过来：精确匹配很稳，但换个说法就空手而归，而且中文分词是个坑——FTS5 默认 tokenizer 对 CJK 基本无效，要么上 trigram，要么外接 jieba。

实际流量是混着的：一半是“上次部署报错怎么解决的”，一半是"`retry_queue` 的默认值是多少"。单一检索器注定有一半场景吃亏。

## 做法：混合检索 + 简单路由

我的最终结构（规模：几千条记忆，单机跑）：

1. **存储**：SQLite 一张 `memories` 表（content、timestamp、project 标签），同库建 FTS5 虚拟表（trigram tokenizer）；向量用 sqlite-vec 存 `bge-small-zh` 的 embedding，入库时生成一次。
2. **双路检索**：每次 recall 同时跑——FTS MATCH 取 top10，向量余弦取 top10。
3. **融合**：用 RRF（Reciprocal Rank Fusion），`score = Σ 1/(60 + rank)`。不要拿原始分直接加权，这是我最先踩的坑（下详）。
4. **路由偏置**：查询里出现错误码 / UUID / 驼峰命名等模式（一个正则就够），把 FTS 路权重调高；纯自然语言问句则偏语义路。
5. **过滤层**在融合之后做：project 标签过滤 + 时间衰减，避免旧项目记忆刷屏。

## 踩坑点

- **量纲不可比**：余弦相似度和 BM25 分数直接加权平均，结果基本随机。必须用 rank 融合（RRF）或各自归一化后再加权。
- **中文分词**：FTS5 的 `unicode61` 对中文约等于按字切，召回一坨噪声，直接换 trigram。
- **embedding 模型**：选中文/多语向的。纯英文模型对中文记忆的语义召回明显偏差，这个差异大到肉眼可见。
- **chunk 别太大**：按对话轮或单条笔记入库。一条塞几千字，向量会被稀释，什么都召不准。
- **精确标识符被淹没**时，加路由偏置比反复调融合权重有效得多。

## 可复用建议

- 顺序上先全文（便宜、精确、可解释），再补语义，不要反过来。
- 给每次 recall 记录命中路径（FTS / vector / both），排障时一目了然。
- 手工攒 20~50 条“查询 → 应命中记忆”的评测集，改任何参数前后各跑一遍 recall@5。数据比感觉靠谱。
- RRF 的 k=60 起步即可，基本不用调。

## 总结

语义搜索解决“换个说法也能找到”，全文搜索解决“一字不差必须找到”，而 memory_recall 的真实查询两类都有。单机规模下，SQLite FTS5 + 本地 embedding + RRF 的组合成本几乎为零，配合一个小评测集迭代就够了。别急着上重量级向量库——先搞清楚你的查询到底长什么样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/23eb601b68408250.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/34f8bf4c4cdd8204.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/57decd0f292d5168.png)

