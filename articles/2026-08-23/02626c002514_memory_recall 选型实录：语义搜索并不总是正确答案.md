---
title: memory_recall 选型实录：语义搜索并不总是正确答案
feedId: 34291
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw/Agent 链路里，`memory_recall` 通常负责从长期记忆库中给当前任务拉取可用上下文。很多实践上来就接 embedding + 向量库，默认“语义搜索更先进”。但在自动化流程里，query 类型很杂：有自然语言问法，也有命令片段、路径、报错码、配置项、任务 ID。语义搜索在这些精确 token 上并不稳，全文搜索反而更可靠。

## 问题

如果只上语义搜索，会出现“我记得存过这个报错，但召回不回来”。比如 `gRPC deadline exceeded` 和 `RPC 超时` 在语义上接近，但对 `/home/opc/.claw/tasks/2024-11-02.json` 这类路径、trace_id、IP、端口，embedding 很难给出稳定相似度。

反过来只上全文搜索，遇到“怎么把任务停掉”和“如何取消运行中的任务”这类同义改写，可能漏召回。`memory_recall` 不是搜索越“聪明”越好，而是召回结果要稳定、可预期、可排查。

## 做法/步骤

我的做法不是二选一，而是先建 baseline，再做双路召回。

### 1. 先盘点记忆类型

把 memory 分成几类：事实记录、命令/配置、路径/ID、自然语言总结、时间线。然后统计真实 query 的分布。精确引用类多，就不要指望纯语义召回。

### 2. 先上全文 baseline

用 SQLite FTS5 足够做起点。中文场景建议用 trigram tokenizer，避免默认分词把“内存泄漏”拆坏：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  memory_id UNINDEXED,
  tokenize = 'trigram'
);
```

查询时按 `bm25(memory_fts)` 排序，保留 top_k。全文搜索对路径、报错码、命令片段很稳，成本也低。

### 3. 再叠加语义召回

用本地 embedding 模型把 memory 切片向量化，query 同样向量化后做余弦相似度。语义召回的目标不是“更准”，而是覆盖同义改写这一类全文覆盖不到的 query。

### 4. 融合不要直接拼分数

BM25 和 cosine 分数分布完全不同，不能直接加权。工程上简单稳定的是 RRF：

```
score(doc) = sum(1 / (k + rank_i))
```

`k` 从 60 起步，rank 来自两路各自 top 20-50。最后按 score 重排。这样不用调两路分数的量纲。

### 5. 可观测和评测

`memory_recall` 输出尽量带上 `source`、`score`、`chunk_id`。我做了 30 条真实 query 小测试集，覆盖“精确路径类”“同义改写类”“命令类”“时间类”，看 Recall@5 和 MRR。结果是 hybrid 明显好于单路，但前提是两路都不太差。

## 踩坑点

- **语义搜索对短 query、路径、ID 不敏感**。不要让它去匹配日志摘要里的 trace_id、时间戳、IP。
- **FTS5 中文分词默认不行**。trigram 能缓解，但索引体积会涨，需要控制 memory 切片大小。
- **向量更新一致性**。memory 内容改了或删了，向量必须同步删/更新。漏更新会出现“召回旧版本”。
- **RRF 的 k 值别盲抄**。k 越小，第一名权重越突出；k 太大退化成简单平均。最好在小测试集上调一下。
- **长文全文召回容易塞噪声**。整段大 chunk 既费 token 又干扰模型。建议切片 300-800 token，并保留重叠。

## 可复用建议

1. 先判断 `memory_recall` 的 query 是自然语言多，还是精确引用多。前者偏语义，后者偏全文。
2. 不要裸上向量库。FTS5 本地基线成本几乎为零，先跑通再升级。
3. 双路召回 + RRF 是混合搜索里性价比最高的起点。
4. 保留 memory 元数据：类型、时间、来源、更新时间。检索时加过滤，例如只召回最近 30 天或只召回“命令类”。
5. 记录每次 recall 失败案例，逐步补规则或同义词，比频繁换大 embedding 模型更有效。

## 总结

语义搜索和全文搜索在 `memory_recall` 里不是谁替代谁。语义解决“意思相近”，全文解决“精确匹配”。在 OpenClaw/Agent 这种 query 类型混杂的场景，先上全文 baseline，再用语义补同义改写，最后 RRF 融合，通常比直接上高端向量库更稳。关键不是模型多强，而是建立小型评测集和可观测输出，否则只能靠体感调参。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/013422f2b76e90d3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/4a6ffe680ad40dd8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/77117e40f4a0e3b0.png)

