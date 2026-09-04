---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索，实测与混合方案
feedId: 36009
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 的 `memory_recall` 是 agent 长期记忆的入口：记忆写入时被切成 chunk 存库，查询时召回 top-k 条注入上下文。召回质量直接决定助手"记不记得住事"——召回错了，后面模型再强也白搭。

常见的两条召回路线：

- **语义搜索**：embedding + 向量检索（sqlite-vec、Qdrant 等），按"意思相近"匹配
- **全文搜索**：FTS5 / BM25，按关键词精确匹配

社区里经常争论哪个"更好"。我拿自己的记忆库做了一轮对比，结论是：这不是二选一的问题。

## 问题设定

测试语料约 1200 条记忆：会议纪要、用户偏好、项目决策、踩坑记录，中英混合。构造 20 组真实查询，人工标注相关记忆，对比两路召回。

- **方案 A（语义）**：`bge-small-zh-v1.5` embedding + sqlite-vec，cosine 相似度取 top-5
- **方案 B（全文）**：SQLite FTS5 + trigram tokenizer 取 top-5

## 实测观察

**语义搜索赢的场景**：换说法的模糊查询。比如"上次说的那个部署翻车的事"，能召回内容完全没重叠词的《生产环境回滚排查记录》。全文搜索在这种查询下基本返回空。

**全文搜索赢的场景**：精确标识符。查 `kubectl rollout undo`、`PROJ-2317`、某个内部命令缩写，全文搜索秒中，语义搜索反而召回一堆"讲部署但不是这条"的近似记忆。

**语义搜索的暗坑**：会召回"看起来像但不是"的记忆，相似度 0.82 的错误答案比 0.65 的正确答案更早注入上下文，模型容易被带偏。

## 混合方案：RRF 融合

两路各取 top-10，用 Reciprocal Rank Fusion 合并后取 top-5：

```python
def rrf(*ranked_lists, k=60):
    scores = {}
    for lst in ranked_lists:
        for rank, doc_id in enumerate(lst):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

核心代码 30 行以内，不需要额外依赖。实测这版在 20 组查询里 17 组命中，明显好于单路。

## 踩坑点

1. **FTS5 默认分词器对中文几乎失效**：unicode61 会把整句中文当成一个 token，搜什么都搜不到。必须换 trigram，或者写入前先 jieba 分词。
2. **embedding 模型不能混用**：写入和查询必须用同一个模型，中途换模型要全量重建向量，否则召回全乱。
3. **阈值调不好就不如不设**：相似度阈值高了召回为空、低了注入垃圾。固定 top-k + 去重更省心。
4. **旧记忆污染**：记忆不清理的话，语义搜索会稳定优先召回过时版本（毕竟和查询最像）。加时间衰减因子，或写入时带版本标记让新条目覆盖旧的。
5. **chunk 长度**：超过 500 字语义被稀释，低于 100 字丢上下文，我落在 200~400 字比较稳。

## 可复用建议

- 记忆量小于 5000 条时，**先跑通 FTS5 + trigram**，零依赖零成本，够用很久
- 查询里出现 ID / 命令 / 专有名词的概率不低，**全文通道必须保留**
- 融合用 RRF，**不要对两路原始分数加权平均**——量纲不同，权重没有意义
- 写入时打好元数据（类型、时间、项目），**过滤先于搜索**，能省一半召回噪声

## 总结

语义搜索解决"换个说法也能找到"，全文搜索解决"精确命中标识符"。`memory_recall` 场景两类查询都会出现，RRF 混合是当前性价比最高的解法。落地顺序建议：先 FTS 跑通基线，再叠加向量，不要一上来就搬重型向量库——多数个人 agent 的记忆库规模，SQLite 一把梭就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c55200f7ccc4a55a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/f0f158a7044f8413.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/41ac354730eaa4ac.png)

