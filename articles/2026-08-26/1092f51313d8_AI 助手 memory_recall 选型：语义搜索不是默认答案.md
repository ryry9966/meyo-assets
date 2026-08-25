---
title: AI 助手 memory_recall 选型：语义搜索不是默认答案
feedId: 34794
source: 综合讨论
publishedAt: 2026-08-26
---

# AI 助手 memory_recall 选型：语义搜索不是默认答案

给 OpenClaw 这类 Agent 做长期记忆时，很多开发者会直接把 memory_recall 做成 embedding + 余弦相似度。我实际在几个插件和 MCP memory server 里验证后，发现这个选择经常被高估。本文记录我在 memory recall 工具上对语义搜索和全文搜索的对比测试，以及可复用的工程结论。

## 背景

memory_recall 的本质不是“存”，而是“在正确的时间把相关记忆拉回来”。它通常作为 MCP 工具或插件动作暴露给模型。查询输入可能来自模型基于当前对话生成的召回 query，而不是用户原话。这一点很关键，因为模型生成的 query 可能是自然语言，也可能包含它从上下文中提取的精确实体。

## 问题

全文搜索（FTS/BM25）强在精确词、路径、命令、ID，弱在同义改写和模糊意图；语义搜索（embedding）强在自然语言、跨语言和意图匹配，弱在精确符号和可解释性。直接选一边都会出问题。

语义搜索会被“uid=8f3a9c”、“nginx-ingress-2024.yaml”、“POST /api/v2/rebuild”这类内容打败，因为这些不是自然语言，向量相似度排序不稳定。全文搜索则会被“上回我说的那种不要重试的方案”这类查询打败，因为没有关键词命中。

## 做法与步骤

### 1. 设计统一的 memory schema

先把 memory 数据模型收敛，避免后续两种检索各自为政：

```sql
CREATE TABLE memory (
  id TEXT PRIMARY KEY,
  actor_id TEXT NOT NULL,
  type TEXT NOT NULL,          -- preference / decision / fact / command
  content TEXT NOT NULL,
  content_norm TEXT NOT NULL,  -- 归一化后的文本
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE VIRTUAL TABLE memory_fts USING fts5(
  content_norm,
  content='memory',
  content_rowid='rowid',
  tokenize='trigram'
);
```

中文场景如果不做专门分词，FTS5 默认 tokenizer 对中文不友好，可以用 `trigram` 先跑通；如果追求更可控，写入 `content_norm` 前用 jieba 等预分词。

### 2. 先建全文搜索基线

写入时归一化：小写、去多余空白、英文数字保留、中文按需分词。查询时执行同样归一化。用真实会话里抽出的 30 条 query 做评测，计算 recall@5。这一步是为了拿到可解释、易复现的底线指标。

### 3. 再上语义检索

本地跑 embedding 模型，例如 bge-small-zh 或 paraphrase-multilingual-MiniLM，生成 384/512 维向量，存 sqlite-vec 或 numpy 矩阵。查询时同样 embedding 后取 cosine topK。不接外部向量 API，降低延迟和隐私风险。

### 4. 对比典型 query

实测中，自然语言偏好类 query（“我之前说过不喜欢什么部署方式”）语义搜索明显更好；精确查询（“上周那个 prod 报错里出现的 container id”）全文搜索更稳。

### 5. 混合检索兜底

两种结果各自取 top10，用 RRF 合并：

```
score_rrf = 1 / (k + rank_fts) + 1 / (k + rank_semantic)
```

固定 `k=60` 即可，不需要复杂学习。离线调参时优先保证精确类 query 的召回。

## 踩坑点

- **FTS 中文分词**：默认 tokenizer 对中文不友好。`trigram` 能解决但索引会变大；预分词更可控，但需要维护词表。
- **向量阈值不通用**：cosine 0.7 不是每个模型都适用。不同 embedding 模型分差很大，阈值必须在自己评测集上校准。
- **ID、路径、版本号容易被 embedding 漏掉**：hash、uuid、文件路径、版本号、API 名称，向量相似度经常排序失败，因为它们的语义信息很弱。这些内容应走全文或精确匹配。
- **更新成本差异大**：FTS 更新即写即生效；向量索引更新需要重新 embedding。频繁更新的 memory 只走 FTS 或延迟重建向量。
- **内容长度会稀释相似度**：过长的 memory embedding 后相似度偏低。写入前需要拆成更小单元，或额外存摘要。但不要只存摘要，否则会丢失可被全文检索的细节。
- **先过滤后检索**：按 `actor_id` 和 `type` 先过滤，再检索。否则权限串味、小库也容易召回无关结果。

## 可复用建议

- **默认 FTS5 打底**。明确记忆、精确事实、命令、路径、ID 优先全文检索，解决大部分真实召回。
- **语义搜索只做补充**。当 FTS 分数低，或者 query 明显是自然语言偏好/意图时，再走语义检索。
- **写入时打 type 标签**。召回时按 `type` 过滤，往往比调整 embedding 或阈值更有效。
- **保留评测集**。每次换模型、调 chunk、改 tokenizer 都重新跑一遍 recall@5 和 MRR，避免回归。
- **几万条以内 SQLite FTS5 足够快**。不需要一上来就上重型向量库，sqlite-vec 足够用。

## 总结

语义搜索擅长模糊意图，全文搜索擅长精确事实。在 OpenClaw、Agent、MCP 这类场景里，成熟的 memory_recall 不是 all in vector，而是：

> 过滤 + 全文检索打底，必要时用语义搜索补充，最后 RRF 合并。

这个组合实现简单、可解释、易维护，也更容易在真实会话中稳定工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/f0e53b5fe62217eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/62041905fcb81608.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/27b69e10af57bcf5.png)

