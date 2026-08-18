---
title: AI 助手的 memory_recall：语义搜索不是银弹，全文检索仍值得留一手
feedId: 33792
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景：memory_recall 的两个路线

在 OpenClaw / Agent / MCP 的自动化实践里，memory_recall 基本绕不开两个选型：

- **全文搜索（FTS）**：关键词命中，快、可解释、好调试。
- **语义搜索**：把 memory 向量化，按“意思”召回，能处理换一种说法的情况。

很多人在给 AI 助手接 memory 时，会默认“上向量库、上 embedding 一定更聪明”。工程上不是这样。尤其当你的 memory 数据量不大、以中文为主、且混着大量命令/报错/配置片段时，语义搜索可能还不如一段靠谱的 FTS5。

本文不讨论“哪个理论更好”，只记录在 OpenClaw 类助手里做 memory_recall 时，两种方案的实际表现和可复用的混合做法。

---

## 问题：单独用某一种，都会丢东西

### 全文搜索的典型问题

SQLite FTS5 默认 tokenizer 对中文不友好。比如 memory 里有：

> “部署脚本报错：MCP server 启动超时”

你搜索“脚本部署报错”可能查不到，因为默认分词没分好。甚至你搜“MCP server”很正常，但搜“mcp 启动问题”就漏了。

这说明：

- 关键词一致时，FTS 又快又准；
- 换个词序、换个说法，FTS 直接哑火。

### 语义搜索的典型问题

语义搜索强在“意思接近”，但有两个工程化痛点：

1. **专有名词、命令、错误码会被过度泛化**  
   查“MCP server config error”，语义模型可能把“OAuth token 配置错误”也排到前面，因为都是“配置类错误”。但你要的就是那个 MCP server 的具体报错。

2. **小数据集上阈值很难设**  
   memory 只有几百到几千条时，余弦相似度 0.7 可能什么都召不回；放宽到 0.5，又混进一堆弱相关记忆。阈值变成玄学。

所以实际工程里，我会保留两路召回：**FTS 负责精确召回，语义负责模糊召回。**

---

## 做法/步骤

### 1. 一条 memory 至少三类存储

我习惯把 memory 拆成：

- `content`：原始文本，保留给 LLM 最终阅读。
- `terms`：经过分词/清洗后的关键词文本，专门给 FTS 用。
- `embedding`：向量，专门给语义召回用。

表结构不复杂：

```sql
CREATE TABLE memory (
  id TEXT PRIMARY KEY,
  content TEXT NOT NULL,
  terms TEXT,
  embedding BLOB,
  metadata TEXT,
  created_at INTEGER,
  updated_at INTEGER
);

CREATE VIRTUAL TABLE memory_fts USING fts5(
  terms,
  content,
  tokenize='trigram'
);
```

这里用 FTS5 的 `trigram` tokenizer，比默认 unicode61 更适合中文子串匹配。虽然 trigram 索引体积大一点，但少了很多分词坑。

### 2. 写入时做两件事

写入一条 memory 时：

- 对 `content` 做一次轻量级中文分词，生成 `terms`；
- 调用 embedding 模型生成向量，存 `embedding`。

对于 OpenClaw / MCP 这类场景，memory 不一定都来自自然语言。很多是命令输出、报错、配置片段。这类文本**不要过度分词**，保留原始 token 更重要。

所以更稳的做法是：

```text
terms = original_content + " " + jieba_lite_segment(content)
```

既保留原串，又补一层分词。查询时也同样处理。

### 3. 召回时两路并行，再融合

查询流程：

1. `query` 同样生成 `query_terms`。
2. FTS 查询：

   ```sql
   SELECT id, bm25(memory_fts) AS score
   FROM memory_fts
   WHERE memory_fts MATCH ?
   ORDER BY score
   LIMIT 20;
   ```

3. 语义查询：拿 `query_embedding` 在向量表里找 top 20。
4. 两路结果用 **RRF（Reciprocal Rank Fusion）** 合并：

   ```text
   RRF_score = SUM(1 / (k + rank))
   ```

   `k` 一般从 60 开始调。

最后取合并后的 top 5～10 条，交给 LLM 作为 memory context。

---

## 踩坑点

### 1. 中文 FTS 别用默认 tokenizer

FTS5 的 `unicode61` 对中文就是按字处理，查询效果很差。`trigram` 虽然要求 SQLite 版本至少 3.34，但当前环境基本都满足。代价是索引更大，写入更慢。对 memory 这种读多写少的场景，完全可接受。

### 2. 语义搜索的相似度阈值必须按业务校准

不要抄一个 0.7 上来就用。建议准备一组真实 query 和 expected memory，跑一遍召回，看 F1 曲线再定阈值。

如果一个 embedding 模型从本地换成 API，或者 memory 数据集从 500 条涨到 5000 条，阈值可能要重新调。

### 3. 错误码、命令、文件名这些字段，需要额外保留精确匹配

这是语义搜索最容易翻车的地方。`ERR_CONN_REFUSED`、`mcp.json`、`/v1/tools`、`redis-6.2.1`，这些字符串不应当只依赖向量。语义模型会把它们当成“看起来像错误/路径”的通用 token。

建议把这些关键字段存在 metadata 里，召回前用 metadata 过滤，或者单独建虚拟表走精确查询。

### 4. 混合召回不是相加，是融合

不要简单地把两路分数归一化后相加。FTS 的 bm25 分数和余弦相似度量纲完全不同。直接相加容易让某一路“淹没”另一路。

RRF 只关心排名，不关心分数大小，工程上稳定得多。

---

## 可复用建议

1. **先上 FTS，再考虑 semantic。**  
   先把“查得到”的问题解决，再解决“查得懂”的问题。很多 memory 需求其实 FTS 就够。

2. **关键字段保留精确入口。**  
   agent_id、tool_name、error_code、file_path、date，这些先过滤，再进混合召回。

3. **建一个 golden set。**  
   不需要很大，30～50 条真实 query 和对应 memory 就行。每次换 embedding 模型、调权重、改分词，都跑一遍，别凭感觉。

4. **控制语义召回范围。**  
   不要一上来全库语义搜索。先按 agent、plugin 或时间窗口做 metadata 过滤，再在子集里做语义召回。这样延迟和噪声都会好很多。

5. **LLM rerank 要克制。**  
   混合召回后接一次 LLM rerank 确实能提升质量，但对 memory_recall 这种高频路径，成本会涨得很快。建议只在 top 候选较多、且敏感任务里启动。

---

## 总结

memory_recall 没有“语义搜索一定更好”的结论。工程上更实用的判断是：

- **精确关键词、报错、命令、配置片段：交给 FTS。**
- **模糊意图、回忆类内容、跨场景联想：交给语义搜索。**
- **两者不是替代，是并行召回后融合。**

在 OpenClaw / Agent / MCP 的实践里，这种“FTS + embedding + RRF”的轻量混合方案，比一上来就重向量库、重 rerank 要稳得多，也更容易持续迭代。

---

