---
title: memory_recall 选型：语义搜索和全文搜索，别二选一
feedId: 34279
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的 Agent 实践里，`memory_recall` 通常负责从历史会话、执行日志、插件返回、MCP 资源里召回相关上下文。早期很多实现会直接上 embedding 做语义搜索，理由是“自然语言查询用向量更顺”。但真实场景里，Agent 发出的查询并不全是自然语言，还混着报错码、版本号、路径、接口名、时间点。此时纯语义搜索会频繁翻车，全文搜索也不一定能完全顶上。

## 问题

语义搜索擅长理解“意思相近”，比如“模型输出太慢”“生成很久才返回”“响应延迟高”可以互相命中。但它对硬标识不稳定：`v1.8.3`、`/tmp/xxx.log`、`ECONNREFUSED`、`uuid:xxxx` 这类 token，embedding 很容易召回一堆不相关内容。

全文搜索正好相反。FTS5/BM25 能精确匹配 `gateway timeout` 或某个版本号，但理解不了“上次那个登录失败的问题”和“鉴权报错”之间的关系。二选一，最终都会在某种查询上丢召回。

所以问题不应该是“语义搜索 vs 全文搜索哪个好”，而是 memory_recall 里怎么组合，以及什么时候用哪个。

## 做法/步骤

我在 OpenClaw 里做了一个最小验证，步骤如下：

1. **准备回归集**  
   从真实使用日志抽 30–50 条查询，标注理想命中的记忆片段。不用很大，但必须包含几类典型查询：自然语言描述、代码符号/报错、时间相关、路径/ID。

2. **建全文索引**  
   用 SQLite FTS5 做 BM25 全文召回。中文内容要特别注意分词，否则 FTS5 容易按整句或单字切分，召回质量很差。可以预分词后写入，或使用支持中文的 tokenizer。

3. **建向量索引**  
   用 `sqlite-vec` 或 `pgvector` 存 embedding。模型用本地 `bge-small-zh` 或 `text-embedding-3-small` 就够，不必上大模型。

4. **对比指标**  
   重点看 `Recall@5`、`MRR`、平均延迟、更新成本。Agent 上下文预算通常有限，看 `Recall@10` 容易虚高，实际能塞进 prompt 的往往只有前 3–5 条。

5. **混合召回**  
   比较实际的方案是：FTS 和语义各召回 20 条，用 RRF 融合后取 top_k。伪代码大概如下：

```python
def hybrid_recall(q):
    fts_hits = fts.search(q, limit=20)
    sem_hits = vec.search(embed(q), limit=20)
    return rrf_fusion(fts_hits, sem_hits, k=60)[:5]
```

也可以先 FTS 做粗召回，再用 embedding/重排模型做二次排序。前者更省资源，后者更适合相关性要求高的场景。

## 踩坑点

- **纯语义对错误码/版本号/UUID/路径召回不稳**  
  日志类 memory 里如果只有向量索引，查 `ECONNREFUSED` 可能召回一堆网络超时但完全不相关的内容。这类查询必须保留 FTS 或结构化过滤。

- **FTS 中文分词不配置等于白建**  
  FTS5 默认 tokenizer 对中文不友好，不处理时要么整段切，要么单字切，召回质量很随机。需要在写入侧统一分词，或使用 ICU/外部分词器。

- **RRF 的 k 值不能无脑抄**  
  固定 `k=60` 只是常见默认值，不一定适合你的数据分布。k 越小，排名靠前的结果越占优势；k 越大，两个来源的分数越平均。需要在回归集上调。

- **embedding 模型换版本后旧向量不能直接用**  
  向量索引需要记录 embedding 模型名和生成时间。换模型后如果不迁移，旧向量和新查询不在同一空间，语义召回会明显劣化。

- **上下文预算有限时，精确比多更重要**  
  宁可返回 3 条高度相关的内容，也不要 8 条模糊内容。OpenClaw 后续推理很受上下文质量影响，召回多但噪声大反而会拉低任务成功率。

## 可复用建议

- **`memory_recall` 暴露 `search_mode`**  
  建议支持 `auto/fts/semantic/hybrid`，并在 debug 信息里返回每条结果的 score 和来源。出问题时能快速判断是哪个召回通道掉链子。

- **先上 FTS 兜底，再做语义扩展**  
  确定性查询走全文；自然语言、偏好、总结类内容走语义。不要一上来全盘 embedding。

- **实体类信息用结构化字段存**  
  版本号、ID、路径、接口名、时间点，直接放结构化字段或走过滤条件，不要靠向量硬搜。

- **维护一个 30–50 条的小回归集**  
  每次改 embedding 模型、分词方式、RRF 权重，都跑一遍。不用做成复杂平台，一个脚本即可。

## 总结

语义搜索解决“意思相近”，全文搜索解决“精确匹配”。在 OpenClaw 的 memory_recall 里，这不是算法选型问题，而是召回策略问题。比较务实的顺序是：**结构化字段存实体 + FTS 做确定性兜底 + 语义搜索做自然语言扩展**。混合检索只是手段，目的是让两类查询都不至于漏掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d6c0ebb5d77a9365.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f4e389d0fa282919.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/2dcdf44c135074c6.png)

