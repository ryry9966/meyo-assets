---
title: OpenClaw memory_recall 实践：语义搜索 vs 全文搜索，先别急着上向量库
feedId: 33915
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 里给 Agent 接上 memory 之后，`memory_recall` 通常是最频繁被调用的隐藏工具之一。它决定了 Agent 在任务开始时能“想起”多少上下文，也直接影响后续工具调用质量。

常见实现路线分两类：一类走 embedding 语义检索，把 memory 片段编码成向量做相似度匹配；另一类走全文检索，比如 SQLite FTS5、BM25 或简单关键词匹配。很多人一上来就选择向量库，觉得语义搜索更智能。但在真实 memory_recall 场景里，这个结论并不总成立。

## 问题

两种搜索的差异在 memory_recall 里非常明显：

- **语义搜索**能理解“我上次怎么解决那个报错的”这类改写查询，但容易“自信地”召回语义相近、实际无关的内容，尤其是 short query。
- **全文搜索**对命令、路径、函数名、配置项等精确匹配很稳定，但怕同义改写、中英文混写和轻微拼写偏差。

哪种更好，取决于 memory 的内容类型、查询方式和更新频率。比起直接选边，更可靠的做法是先做基线对比，再决定策略。

## 做法/步骤

我在 OpenClaw 插件里把 memory 分了两层：一层是结构化记录，例如任务日志、决定、偏好；另一层是代码片段、命令、路径等精确引用。

### 1. 先跑全文检索基线

用 SQLite FTS5 给 memory 表建索引，先不引入任何向量逻辑：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(content, tokenize = 'unicode61');
```

配置 `memory_recall` 走 FTS5，`top_k=5`，记录命中结果、耗时和失败查询。

### 2. 再跑语义检索基线

本地 embedding 用 `bge-m3` 或 `bge-small-zh`，向量存储可以直接用 SQLite + `sqlite-vec`，避免额外起服务。对同一批查询返回 `top_k=5`，余弦相似度阈值从 0.5 开始调。

### 3. 用真实查询集对比

构造 50 条实际使用中会出现的查询，人工标注相关 memory。对比两个方案的 `recall@5`、`precision@5`、p50/p95 延迟。我的结果里，精确类查询 FTS5 明显更好；模糊意图查询语义检索更好，但误召回也更高。

### 4. 按结果选择或混合

最后我没有二选一，而是做了一个轻量 search router：

```python
def recall(query):
    fts_hits = fts5.search(query, limit=20)
    if fts_hits:
        emb = embed(query)
        scored = sorted(
            fts_hits,
            key=lambda r: cosine(emb, r.embedding),
            reverse=True
        )
        return [r for r in scored if score > threshold][:5]
    return semantic_search(query)
```

也就是先全文召回候选，再用 embedding 重排。这样精确查询不会漏，模糊查询也能借上语义信号。

## 踩坑点

- **语义搜索阈值不能照搬 0.7**。中文短查询相似度普遍偏低，0.7 几乎召不回；建议先跑一批查询看相似度分布，从 0.5 左右开始调。
- **embedding 模型语种不匹配**。用英文模型存中文 memory，语义搜索基本失效；换多语言模型后延迟又会上升，需要权衡。
- **FTS5 默认分词对中文不友好**。`unicode61` 对中文按单字处理，短词召回差。可以换 `trigram` tokenizer，或者在写入前做一层 jieba 分词。
- **混合检索顺序很重要**。先语义后全文，可能漏掉精确命令、ID、路径；先全文后语义，对改写查询更稳。
- **memory 更新后向量没重建**。写入 memory 时最好同步生成 embedding，否则语义索引会越来越脏，召回结果会“穿越”。
- **长 memory 整段写入**会让相似度漂移。尽量按条目或片段切分，不要把一个完整对话摘要存成一大段。

## 可复用建议

- **按内容类型分治**：命令、路径、API key、函数名走全文/精确匹配；偏好、总结、任务背景走语义。
- **小规模 memory 先上 FTS5**。如果 memory 总量在几千条以内，向量库不一定有必要，FTS5 基线足够排查大部分问题。
- **保留可观测性**：给 `memory_recall` 返回结果加上 `source=fts|semantic|hybrid` 和 score，方便后期定位误召回。
- **阈值要按业务数据采样校准**，不要固定成一个“通用值”。
- **如果用 MCP memory server**，先确认它的 recall 是否支持切换 search type；不支持的话，在插件层包一个 search router，不要直接绑定某个实现。

## 总结

语义搜索和全文搜索不是谁替代谁，而是不同召回策略的取舍。工程上更务实的路线是：全文检索做精确基线，语义检索补改写和模糊意图，最后用混合重排与阈值校准兜底。

OpenClaw 里的 `memory_recall` 应该是一个可配置的 search router，而不是绑定单一检索实现。先做基线、再上混合，通常比直接引入向量库更省事，也更可解释。

---

