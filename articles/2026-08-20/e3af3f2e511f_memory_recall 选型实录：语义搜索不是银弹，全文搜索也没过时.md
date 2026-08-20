---
title: memory_recall 选型实录：语义搜索不是银弹，全文搜索也没过时
feedId: 33910
source: 综合讨论
publishedAt: 2026-08-20
---

在 OpenClaw 里给 Agent 做长期记忆时，`memory_recall` 基本绕不开两条路线：把记忆向量化后做语义搜索，或者用 SQLite FTS5/ES 做全文搜索。我们团队一开始也倾向“语义搜索更智能”，但实际跑了几轮插件和 MCP 工具后，结论更接近工程判断：**两者不是替代关系，更适合当两条召回通道。**

## 背景

Agent 的记忆会混着多种内容：任务日志、用户偏好、代码片段、报错记录、决策结论。比如一条记忆可能是：

> `orders/sync_worker` 偶发 HTTP 504，增加了 3 次重试，间隔 1s/5s/15s。

当用户问“下单同步任务失败怎么处理”时，语义搜索能靠 paraphrase 找到这条。但如果问“sync_worker 的 504 配置”，语义搜索常常会漏，因为它不擅长处理短标识符、错误码、版本号、参数名。反过来，全文搜索能精确命中 `sync_worker` 和 `504`，但对“订单同步服务超时”这种说法又容易失配。

所以问题不是“哪个更好”，而是：**你的 memory 里哪类查询占比更高，以及如何把两者放进同一个 recall 工具里。**

## 做法

我们给 OpenClaw 的 memory 插件做了一个可对比的 `search_memory` 工具，后端同时支持 FTS5 和向量检索。

### 1. 建一个统一记忆表

先把记忆落到 SQLite，保留来源和元数据：

```sql
CREATE TABLE memory (
  id INTEGER PRIMARY KEY,
  content TEXT NOT NULL,
  type TEXT,
  created_at TEXT,
  meta TEXT
);
```

全文索引可以直接用 FTS5 外部内容表：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  content_rowid='id',
  content='memory',
  tokenize='trigram'
);
```

中文场景建议用 `trigram`。SQLite 默认 `unicode61` 对中文分词不友好，会退化成整句匹配或者错误切分，召回质量下降很明显。

### 2. 向量通道单独建

向量库我们用了 sqlite-vec，和 SQLite 放一起方便管理。写入时先 chunk，再调 embedding 模型生成向量。chunk 大小建议 256–512 token，太小语义不完整，太大召回时噪音多。

关键约束：**embedding 模型必须固定版本，chunk 策略变更后旧向量要计划重建。** 否则同一批记忆里混着不同维度或不同语义空间的向量，检索结果会很飘。

### 3. 暴露 MCP 工具

插件里提供一个 MCP 工具：

```
search_memory(query, mode: semantic|fts|hybrid, limit, filters)
```

返回结构带 `id`、`content`、`score`、`mode`、`source`。`source` 和 `score` 很重要，便于 Agent 后续决策和人工排查。

### 4. 建固定评测集

我们做了一个很小的评测集，覆盖四类 query：

- 精确实体：找 `sync_worker`、`504`、某个版本号
- 同义改写：把“超时”说成“卡住”“无响应”
- 混合场景：既有自然语言，又带实体名
- 时间/范围过滤：只要最近 7 天的记忆

统一跑 `recall@5`，不要只看 top1。因为 Agent 场景里返回 5 条给下游做上下文，比只赌第一名更接近真实使用。

### 5. 混合检索

两个通道各自召回 top_k，然后用 RRF 融合：

```
score(doc) = sum(1 / (60 + rank_in_channel))
```

不要直接把余弦相似度和 BM25 分数加在一起，量纲不同，直接加权会被某一通道主导。工程上 RRF 最省事，也稳定。

## 踩坑点

1. **中文 FTS5 分词**：`unicode61` 基本不可用，上 `trigram` 或外部分词器。
2. **元数据过滤被忽略**：向量库先过滤时间、用户、会话范围，再做语义搜索，否则经常召回跨用户或过期记忆。
3. **只优化单一指标**：语义搜索 top1 高，但精确实体召回差，要分开统计。
4. **embedding 模型不锁定**：换模型后维度不一致，旧向量全部作废。
5. **score 跨后端不可比**：不要拿 BM25 score 和余弦相似度直接比较，用 RRF。
6. **chunk 策略影响召回**：chunk 太大，返回内容过于宽泛；太小，语义碎片化。

## 可复用建议

- 如果记忆里精确标识符、命令、报错码占比高，**默认走 FTS 或 hybrid**。
- 大量自然语言任务记录、用户偏好，语义通道才更占优势。
- 所有查询先做 `metadata filters`，再召回。
- 工具接口不要写死模式，`search_memory` 支持 `mode` 和 `filters`。
- 建立一个固定评测集，升级 embedding、chunk、融合策略后回归。
- 保留 `source` 和 `score`，防止 Agent 拿着不可解释的分数乱引用。

## 总结

对 OpenClaw 的 `memory_recall` 来说，语义搜索适合理解意图，全文搜索适合精确实体。工程上更稳妥的做法是：**先过滤、双通道召回、RRF 融合、按需 rerank。** 不要因为“语义搜索”听起来更智能，就把 FTS5 从记忆系统里删掉。真正好用的记忆召回，不是选一个更好的算法，而是让不同算法各干各的活。

---

