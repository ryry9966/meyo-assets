---
title: memory_recall 实测：语义搜索 vs 全文搜索，别急着上向量库
feedId: 36697
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

Agent 跑久了，memory 里会积累大量条目：会话摘要、用户偏好、踩过的坑、项目决策。`memory_recall` 的检索质量直接决定 agent 是"记得住事"还是"每次都装失忆"。社区里常见的争论是：上向量语义检索，还是用全文检索（BM25 / FTS5）就够了。

## 问题

我拿约 2000 条真实 memory 条目做了对照测试，查询分两类：

- **模糊意图类**："我之前是不是说过数据库怎么部署来着"
- **精确标识类**："上次 ECONNRESET 报错怎么解决的"

对比三个方案：A）embedding + 余弦相似度（本地模型）；B）SQLite FTS5 + BM25；C）混合召回 + RRF 融合。

## 结论先行

- 模糊意图类查询：**语义检索明显更好**，能命中"换了说法"的旧记忆。
- 精确标识类查询：**全文检索几乎全胜**。报错码、函数名、配置项这类 token，embedding 会把语义稀释，语义检索经常漏召。
- 混合方案两者都稳，实现成本不高。

## 做法

1. memory 存 SQLite，正文列挂 FTS5 索引。
2. 另存一张 embedding 表，写入时用本地模型向量化。
3. recall 时两路并行：FTS5 取 top-10，向量取 top-10。
4. 用 RRF（Reciprocal Rank Fusion）合并：`score = Σ 1/(60 + rank)`，取 top-5 带时间戳喂给模型。

RRF 只看排名不算分值，两路分数不可比的问题直接消失，代码不到 20 行。

## 踩坑点

1. **中文分词**：FTS5 默认 tokenizer 对中文基本失效（整句当一个 token）。要么接 jieba 分词后写入，要么用 trigram tokenizer。
2. **小语料下语义检索优势不大**：几百条以内，BM25 加几个同义词扩展就够了，向量库属于过度设计。
3. **精确标识符是 embedding 的死穴**：`ECONNRESET`、`maxRetries=3` 这类内容建议单独走全文通道，别指望语义救场。
4. **换 embedding 模型 = 全量重建**：向量维度和语义空间都变了，旧索引必须重算，评估好迁移成本再选模型。
5. **别丢时间信息**：三个月前的偏好和昨天的偏好权重不该一样，简单的时间衰减因子效果就不错。

## 可复用建议

- memory 条目少、查询偏精确：**先上 FTS5**，别先建向量库。
- 有明显的"换了说法找不到"痛点：**加一路向量，RRF 融合**。
- 喂给模型的条数控制在 3–5 条，附时间戳，宁少勿杂。
- 两路都召不回时，降级返回最近 N 条 memory，比返回空强。

## 总结

这不是二选一的问题，而是**按查询类型分流**的问题：模糊意图交给语义，精确标识交给全文，RRF 兜底合并。如果你的 agent 刚起步，从 FTS5 开始是最务实的路径——先把分词做好，等真遇到召回缺口再补向量这一路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a18df6dd8c795bcb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/15dd3aa95c23f002.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/eb9f19ec7980c17c.png)

