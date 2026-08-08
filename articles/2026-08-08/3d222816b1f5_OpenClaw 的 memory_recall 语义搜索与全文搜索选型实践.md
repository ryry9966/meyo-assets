---
title: OpenClaw 的 memory_recall 语义搜索与全文搜索选型实践
feedId: 32101
source: 综合讨论
publishedAt: 2026-08-08
---

## 为什么开始纠结 memory_recall

给 OpenClaw 挂上 memory 插件以后，Agent 总算能记住“上周那个导出逻辑为什么要改成异步”。但用着用着就发现一个尴尬的现象：问“帮我把上回那个 CSV 导出的脚本再跑一次”，它带回了一段关于“数据清洗”的笔记，完全不是一回事。问题出在 memory_recall 的检索层——到底该用语义搜索（向量相似度）还是全文搜索（BM25/关键词）？这篇文章记录我在 OpenClaw 体系里同时落地两种方案、踩坑，以及最后形成的一套可复用的混合策略。

**用到的环境**：OpenClaw 0.9.x、插件 openclaw-memory、mcp-server-memory（local vector store + sqlite）、嵌入模型 BAAI/bge-small-zh。全文搜索侧使用内置 jieba 分词 + BM25 实现。

## 语义搜索 vs 全文搜索：问题拆解

先说结论层面的差异：

- **语义搜索**：把记忆片段编码为向量，检索时用余弦相似度召回 top-k。对口语化表达、同义改写非常友好。比如“上次那个并发写入文件出错的方案”可以匹配到“多线程同时写 log 导致覆盖”的历史记录。
- **全文搜索**：基于分词后的倒排索引，匹配严格。适合精确术语、代码片段、ID、命令参数。例如搜索“feat: async export csv”，全文搜索可以精确命中，而语义搜索可能会被 embed 模型误解为“数据导出功能”，带回一堆毫不相干的片段。

实践中暴露的真实痛点：
1. Agent 的 prompt 里混合了自然语言和结构化线索（文件名、错误码、函数名），单一检索必然丢信息。
2. 中文语境下，embedding 模型对混合中英文、特殊符号的泛化能力有限，纯语义召回容易出现“似乎相关但其实无关”的结果，浪费上下文窗口。
3. 全文搜索对“上周”这种时间性描述完全无力，而语义搜索可以理解时间模糊表达。

所以真正的题目不是“哪个好”，而是“在 OpenClaw 的 memory_recall 管道里，如何让两者配合工作”。

## 实践步骤：在 OpenClaw 中配置混合召回

OpenClaw 的 memory 插件允许自定义 `recall_strategy`，配置入口在 plugin manifest 或 memory 配置文件。以下是我使用的双路召回 + 精排方案。

### 1. 索引阶段：双通道写入
在 `openclaw-memory` 的配置中，指定两类索引器：

```yaml
memory:
  store:
    vector:
      enabled: true
      model: BAAI/bge-small-zh
      dim: 512
      metric: cosine
    fulltext:
      enabled: true
      tokenizer: jieba
      language: zh
      store_raw: true   # 保留原文，便于精排时使用高亮
  index_on: new_memory  # 每当新记忆写入时自动索引
```

注意：向量索引和全文索引写入同一逻辑记忆的 ID，这样可以做到结果对齐。

### 2. 检索阶段：混合召回 + RRF 融合
在 memory recall 配置中使用 `fusion` 策略：

```yaml
recall:
  strategy: hybrid
  retriever:
    vector:
      top_k: 15
    fulltext:
      top_k: 15
  fusion:
    method: rrf  # Reciprocal Rank Fusion
    k: 60        # RRF 平滑参数
  final_top_k: 5
```

RRF 的计算成本极低，不需要额外模型，适合 Agent 这种对延迟敏感的场景。经过调参，`k=60` 在中文混合召回中多次实验表现稳定。

### 3. 可选精排（Rerank）
如果对准确率要求更高（例如处理长记忆链），可以开启基于 Cross-encoder 的精排：

```yaml
rerank:
  enabled: false  # 默认关闭，因为 bge-reranker-base 会增加约 200ms 延迟
  model: BAAI/bge-reranker-base
  top_n: 3
```

在工具类 Agent（代码、命令行）场景下，我保持关闭，因为 RRF 融合后的 top5 已经够用，并且延迟代价不值得。

## 踩坑记录

**坑1：向量维度不一致导致召回异常**  
切换嵌入模型后忘记重建索引，512 维的旧向量和 768 维的新查询做余弦相似度时，系统不报错直接返回垃圾结果。解决：每次更改 embedding 模型，必须强制重建向量索引。在 openclaw-memory 中实现一个 `--rebuild-index` 的 CLI 命令或调用 admin API。

**坑2：中文分词器对英文标点和驼峰命名的切割**  
默认 jieba 分词器会把 `getCsvExportCmd` 切成 `get`, `Csv`, `Export`, `Cmd`，导致全文搜索无法精确匹配。建议在 fulltext 配置中增加自定义词典，或者对含下划线、驼峰的字符串进行预处理（如保留原始 token 额外索引一份）。我最终的做法是：识别出代码 token 后，在全文索引字段里增加一个 `raw_code` 子字段，用 whitespace tokenizer 索引。

**坑3：语义搜索的“幻觉匹配”**  
对于简短查询（“fix”），向量检索全部返回句子中包含“修复”的记忆，但其中一半是无关的。对策：设定最小查询长度阈值（<3 词），此时只走全文搜索，避免语义泛化。

**坑4：增量索引下的一致性**  
Agent 更新记忆时（如合并知识片段），全文索引更新了但向量索引却因为异步写队列延迟而未生效。解决方法：openclaw-memory 需要保证索引更新的事务性，或者至少提供 `flush` 端点。临时方案是设置 `sync_index: true`，虽然牺牲少量写入吞吐，但规避了大量调试成本。

## 可复用建议

1. **分类场景，选择主路**：如果你的 OpenClaw 主要做代码助手、命令行操作，优先保证全文搜索的精确性，语义搜索作为补充。反之如果是知识问答或写作助手，语义搜索为主。
2. **监控召回质量**：在 memory 插件中开启 debug 日志，记录每次 recall 的查询、两条路的 candidate 数量、融合后的 top_k。观察一段时间后你就能看出哪个通道在特定查询下贡献了噪声。
3. **利用 MCP 接口进行 A/B 测试**：通过 mcp-server-memory 暴露 `recall` 工具，编写简单的测试脚本，用一批标记好的 query-理想记忆对评估不同策略的 Recall@5。不要跟着感觉走。
4. **轻量化嵌入模型**：部署在本地或边缘时，使用 bge-small-zh (0.13B) 足以应对大部分中文记忆的语义检索，不必追求大模型。内存和首 token 延迟更友好。
5. **保留原文并高亮命中词**：无论最终如何融合，始终在返回的记忆片段中附带全文搜索的命中高亮信息，这对提升 Agent 的可信度非常重要——让用户 (或下游 prompt) 知晓究竟是哪个关键词触发了召回。

## 总结

没有一种召回方案能通吃所有 memory 场景。语义搜索让 Agent 更“懂你”，全文搜索让它更“准确”。在 OpenClaw 的 memory_recall 实践中，双路召回 + RRF 融合是低成本的务实选择；踩过的坑主要集中在分词、向量一致性和索引事务性几个工程细节上。希望这篇记录能帮你在配置自己的 memory 插件时少走弯路，也欢迎在社区交流你的混合检索调参经验。

---

