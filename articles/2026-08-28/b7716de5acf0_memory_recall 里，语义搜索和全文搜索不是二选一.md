---
title: memory_recall 里，语义搜索和全文搜索不是二选一
feedId: 34966
source: 综合讨论
publishedAt: 2026-08-28
---

# memory_recall 里，语义搜索和全文搜索不是二选一

## 背景

最近在 OpenClaw 的 memory 插件里加了一个 memory_recall 工具，通过 MCP 暴露给 Agent 调用。最初为了省事直接上向量语义搜索，结果 Agent 在回忆“服务端口改成 8080”“把缓存路径切到 /data/cache”这类事实时经常找错。后来补了 SQLite FTS5 全文索引做混合召回，漏召回明显减少。

这篇文章不讨论哪种检索更高端，只记录在一个真实 Agent 记忆召回场景里，语义搜索和全文搜索的差异、组合方式和工程坑。

## 问题

语义搜索的好处是能处理同义改写。比如用户问“怎么发布版本”，它能召回“release checklist”相关记忆。但它对数字、命令、路径、专有名词不敏感，且容易把“看起来相近但上下文不同”的旧记忆排到前面。排序分也不容易解释：为什么这条排在前面，经常只能说“相似度 0.73”。

全文搜索（SQLite FTS5）正好相反。它精确、快、可解释，关键词命中就是命中。但遇到同义改写、错别字、跨语言表达时容易漏召回。而且 FTS5 默认分词器对中文不友好。

在 memory_recall 这种场景里，Agent 需要的是“该记得的精确信息不漏”，其次才是“相关意思尽量覆盖”。所以我的结论是：全文搜索保底，语义搜索增强，不要二选一。

## 做法/步骤

### 1. 存储层

记忆表保留原文，并增加结构化字段：

```sql
CREATE TABLE memories (
  id INTEGER PRIMARY KEY,
  content TEXT NOT NULL,
  type TEXT,
  active INTEGER DEFAULT 1,
  importance REAL DEFAULT 0.5,
  created_at TEXT,
  updated_at TEXT
);
```

中文全文索引使用 trigram tokenizer，避免默认 unicode61 把整句当成一个词：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  tokenize='trigram'
);
```

向量可以单独放表，增加 model_version 字段。小规模数据用 sqlite-vec 或直接内存里 numpy 算余弦相似度都行。

### 2. 写入路径

save_memory 时做三件事：

- 保留原始 content；
- 清洗后写入 memory_fts；
- 生成 embedding 并记录模型版本。

建议清洗时只去除 Markdown 符号、合并多余空白，不要改动数字、路径和命令。

### 3. 召回路径

memory_recall 执行顺序固定为：

1. 先结构化过滤：active=1、type、更新时间窗口；
2. FTS5 取 top_k * 2，使用 BM25；
3. 向量取 top_k * 2，按余弦相似度，过滤掉低于 min_sim 的结果；
4. 用 RRF 融合：score = Σ 1 / (60 + rank)；
5. 按融合分截断 top_k，并保留 recall_source 字段，标明来自 fulltext、semantic 或 both。

每次召回记录 query、候选分数、最终条目，写入 audit 表。没有日志的 memory_recall 很难排障。

## 踩坑点

- **中文全文搜索不能开箱即用**。FTS5 默认 unicode61 对中文分词效果很差，必须显式指定 trigram 或先接分词器再写入索引。
- **只存向量不存原文**。排障时完全不知道召回的依据是什么，也影响后续重建。
- **向量模型切换后没有版本隔离**。新旧 embedding 混在一起排序，分数不可比，必须加 model_version 并在查询时过滤或全量重建。
- **固定相似度阈值容易失效**。不同 query 的相似度分布不同，建议使用 top K 截断 + 最小相似度兜底，例如 top_k 8、sim >= 0.55，低于阈值就放弃语义分支。
- **精确 token 漏召回**。端口、版本号、路径、命令这些内容，语义搜索天然不占优势，不要抱太大期望，交给全文搜索。

## 可复用建议

- 先上全文搜索，再叠加向量。FTS5 本地依赖少、速度快、可控，适合作为保底方案。
- 用 RRF 融合，不要直接拼 BM25 和相似度分，因为尺度完全不同。
- 给 memory 增加 active、type、importance、last_accessed_at 等字段，在召回前做结构化过滤，比调相似度阈值更有效。
- 中文场景优先用 trigram tokenizer，省去额外分词服务。
- 准备一个小评测集，覆盖同义改写、精确编号、中英混合、旧记忆干扰四类 query，每次调召回参数都跑一遍。

## 总结

全文搜索负责“精确信息不能漏”，语义搜索负责“意思相近也能找到”。在 Agent 长期记忆里，漏召回比多召回更容易破坏体验，所以我建议 FTS5 全文保底、向量语义增强，再用结构化过滤和 RRF 把两条路合起来。混合召回会增加一点复杂度，但对 memory_recall 这种低频但关键的工具来说，这层复杂度是值得的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ce9c44b8bd72fd37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/217371bd566e6923.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/6f22eedc597f373a.png)

