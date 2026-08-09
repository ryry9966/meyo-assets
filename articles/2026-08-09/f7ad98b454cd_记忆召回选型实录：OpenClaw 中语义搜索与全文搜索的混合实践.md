---
title: 记忆召回选型实录：OpenClaw 中语义搜索与全文搜索的混合实践
feedId: 32251
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：当记忆库开始膨胀

在基于 OpenClaw 构建的 Agent 自动化管线里，Memory 模块不再只是“记住上一轮对话”。我们经常要给助手赋予长期记忆，让它能回顾几周前用户提交的工单、调试过的配置，甚至某次特定场景下的决策依据。实现这个能力的关键就是 `memory_recall`：从数百条历史记忆里，精准捞出当前上下文最相关的那几条。

问题很快浮现——到底该用纯语义搜索，还是老老实实上全文索引？两种思路在早期原型里都被我们试了一遍，结果都不够理想。

- **纯语义搜索**（cosine similarity on embeddings）：用户说“上次那个 Redis 超时的处理办法”，它能兜住“超时”和“timeout”这种同义表达，但时常召回一堆含有“超时”概念却不相关的记忆（例如 Web 请求超时），而真正提到“Redis OOM”的记忆却因为语义距离不够近被淹没了。
- **纯全文搜索**（BM25 / FTS5）：关键字匹配精准无比，但用户随口一句“之前那个缓存雪崩的修法”，如果历史记忆中用的是“cache breakdown”，全文搜索就颗粒无收——因为它理解不了“雪崩”≈“breakdown”。

于是我们开始在 OpenClaw 的 MemoryProvider 插件里引入混合召回，把全文的精确度和向量的语义泛化能力结合起来。本文记录了这个过程的选型、落地步骤和那些折磨人的细节。

## 问题拆解：为什么需要混合召回

从工程角度，记忆召回本质是一个 **信息检索问题**：给定用户查询，从记忆文档库中返回 top-K 相关结果。OpenClaw 允许通过 `MemoryProvider` 抽象后端，MCP 生态里也有可对接的 memory service，但核心召回逻辑还是自己掌控。

混合召回要解决两个层次的问题：
1. 怎么同时从两个异源索引（全文索引 & 向量索引）拿结果。
2. 怎么把两路结果合并成最终的 top-K 列表，而不是简单去重。

我们最终选择了 **Reciprocal Rank Fusion (RRF)**，它不依赖原始打分，只利用排序位置做融合，天然适合 BM25 打分和余弦相似度这种尺度不同的得分。

## 落地步骤

### 1. 扩展 OpenClaw 的 MemoryProvider
在 OpenClaw 配置里注册自定义的 `HybridMemoryProvider`，它内部同时维护一个 SQLite FTS5 表和一个 Chroma 向量集合。

```python
class HybridMemoryProvider(MemoryProvider):
    def __init__(self, config: MemoryConfig):
        self.fts = SQLiteFTSIndex(config.db_path)
        self.vector_store = ChromaVectorStore(config.chroma_path, config.embedding_fn)
```

### 2. 构建双索引
每条记忆写入时，同时：
- 插入到 FTS5 表，字段为记忆内容的纯文本清洗版本（去除 Markdown 标记，保留关键实体）。
- 调用同一个 embedding 接口生成 384 维向量（使用 `all-MiniLM-L6-v2` 的中文微调版），存入 Chroma。

```python
def _index(self, memory: Memory):
    text = sanitize_text(memory.content)
    self.fts.insert(memory.id, text)
    embedding = self.embedding_fn(text)
    self.vector_store.add(memory.id, embedding, metadata)
```

### 3. 实现融合召回
`recall` 方法并行查询两个索引，分别取前 20 条结果，然后用 RRF 合并出最终的 Top-K。

```python
def recall(self, query: str, k: int = 5):
    fts_results = self.fts.search(query, limit=20)   # (id, rank)
    vector_results = self.vector_store.search(query, limit=20)
    merged = reciprocal_rank_fusion(fts_results, vector_results, k=60)
    return [self._fetch_memory(id) for id, score in merged[:k]]
```

RRF 的核心计算异常简单：
```python
score = sum(1.0 / (rank + k) for rank in item_ranks)
```
其中 `k` 常数控制平滑度（我们固定为 60）。

### 4. 接入 Agent 的决策链
将 `recall` 出的记忆文本注入到 Agent 的 system prompt 或 MCP 工具的描述里即可。OpenClaw 已内置对 `memory_recall` 函数的自动调用，只需把 provider 挂上去。

## 踩坑与排障

### 中文全文搜索的分词噩梦
SQLite FTS5 默认按 Unicode 断词，对中文几乎不可用。必须加载自定义的分词器（用 `jieba` 在写入前做分词，再以空格分隔存储），否则“缓存雪崩”会被拆成单字，匹配精度极低。另一个坑是 FTS5 对特殊符号（如 `<>`）的索引策略，需要用 `tokenize=unicode61 "tokenchars=-_"` 来保留部分符号，避免“Redis-哨兵”无法命中。

### 向量模型的维度与速度权衡
为了提高相似度计算效率，我们曾想换用高维模型（768 维），结果 Chroma 的 recall 延迟从 15ms 涨到 50ms，在每次用户对话都触发 recall 的场景下明显拖慢首响速度。最终坚持 384 维，通过精细的 prompt 工程弥补语义精度损失。

### 混合权重不是一劳永逸
不同业务场景下，全文和向量的贡献度差异巨大。例如在强调精确关键词的“服务器报错日志”回忆场景中，全文检索的权重需要更高，否则向量会把一堆“报错但类型不同”的记忆塞进来。我们在 `memory_recall` 工具的参数里增加了 `search_mode` （`exact` / `fuzzy` / `auto`），让调用方可以按需偏置融合权重。

### 增量索引碎片化
频繁追加记忆会导致 SQLite FTS 表出现碎片，查询逐渐变慢。必须定期执行 `INSERT INTO memory_fts(memory_fts) VALUES('optimize')` 来重建倒排结构。我们通过 OpenClaw 的后台定时任务钩子实现了每天凌晨的自动优化。

## 可复用建议

- **从轻量方案起步**：SQLite FTS5 + Chroma + 384 维 embedding 足以支撑 10 万条记忆的毫秒级召回，无需一上来就上 Elasticsearch。
- **为召回加一道缓存**：对同一查询的结果做短期缓存（如 Redis TTL 5 分钟），能大幅降低高频问题下的重复计算。
- **监控 recall 命中率**：在日志里统计 `recall` 调用后用户是否有二次修正提问，间接评估召回质量，再调整融合策略。
- **提供降级开关**：当向量索引不可用时，自动降级为纯全文搜索，保证核心功能不雪崩。

## 总结

在 OpenClaw 这类 Agent 框架里做 memory recall，语义搜索和全文搜索不是非此即彼的替代关系，而是互补组合。纯语义像“理解意图”，纯全文像“精确查找”，混合它们并加上工程化的权重调控，才能让助理在面对模糊回忆和精准复现两种需求时都表现得体面。

最终代码逻辑不超过 200 行，但真正的成本在于持续观察、调整和接受“没有一劳永逸的召回方案”这个事实。

---

