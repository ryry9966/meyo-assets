---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索哪个好
feedId: 31818
source: 综合讨论
publishedAt: 2026-08-06
---

# 为什么 memory_recall 会成为瓶颈

在给 OpenClaw agent 配 memory 插件时，大多数人第一反应是：“存起来就行，反正以后问到了再查”。等到用户说“上次那个方案后来我改过需求，你还记得吗”，agent 翻出 3 条完全不相关的历史，对话体验瞬间崩盘。

memory 不只是“记不记得住”，而是“能不能在需要时把对的那条找出来”。OpenClaw 的 memory_recall 模块正是解决这个问题的关键环节，而它的核心选择是两个检索范式：**语义搜索**和**全文搜索**。

二者目标相同，但对真实场景的适应性差异巨大。如果你的 agent 还会通过 MCP 调用外部工具、拼接多步推理，那 retrieval 质量直接决定链路成败。

---

# 两种召回的实际表现

## 全文搜索：精准但脆弱

全文搜索依赖倒排索引，用分词后的 term 做匹配。配置最简单，Elasticsearch、Meilisearch、甚至 SQLite FTS5 都能接。

优点：
- 对专有名词、订单号、时间表达非常精准；
- 索引更新快，不需要额外编码模型；
- 可解释性强，debug 容易。

缺点同样明显：
- **中文分词塌方**：“笔记本”拆成“笔记”和“本”，搜索“笔记本电脑”可能漏掉记录里的“笔记本”；
- **同义词盲区**：用户问“退订”，memory 里写的是“取消订阅”，全文搜索大概率 0 命中；
- **长文本噪声**：用户复述大段对话，if-idf 把高频日常词放大，冷僻但有信息量的词被淹没。

在工程中踩过最深的坑是：memory 存储了 agent 执行的完整 tool_call 日志，里面夹杂大量 JSON。全文检索时，JSON 的 key “action”、“params” 成为高频干扰，真正关键的行为描述反而排不进前 10。

## 语义搜索：理解力强但维护成本高

语义搜索用 embedding 模型将文本转为向量，做相似度计算。OpenClaw 的 memory 插件中常对接 Chroma、Milvus 或 pgvector。

它的优势是：
- “退订”和“取消订阅”能对上；
- 可以召回意思相近但用词完全不同的记忆；
- 省去手工同义词表。

生产环境的麻烦集中在 3 个地方：
1. **embedding 模型选型影响一切**。轻量模型（如 all-MiniLM-L6-v2）在中文专业领域歧义严重，会把“重启服务”和“重启人生”算成 0.85 相似度，agent 顺着错误记忆执行，后果严重。
2. **召准率不可兼得**。放宽 top_k 会混入弱相关记忆，拉高 token 消耗，导致后续 LLM 推理被噪声淹没；收紧 top_k 又可能漏掉正确答案。
3. **冷启动与增量更新**。每次新增记忆需要 embed + insert，如果选择 local 模型，CPU 负载会暴涨；用 API，费用和延迟都要算账。

# 工程化对比步骤

在 OpenClaw 的 memory 配置中，通常会暴露一个 `retrieval` 块，让用户选择 `type: fulltext` 或 `type: semantic`。以最小可复现实验为例：

**全文检索流程：**
1. 配置 connector，如 `backend: elasticsearch` 或 `backend: sqlite_fts`；
2. 设定 index field 和 search field，明确 analyzer（中文要用 ik_smart 或 jieba）；
3. 在 agent 的 recall step 中，根据 query 拼装 DSL/query 语句，返回匹配 memory；
4. 实测时，对用户 query 做前置同义词扩展（用 LLM 改写一次），可以有效弥补分词缺陷。但要注意，改写步骤会引入额外延迟和调用成本。

**语义检索流程：**
1. 选定 embedding 模型并在 memory plugin 内注册；
2. 使用向量数据库（local 推荐 Chroma，生产可切换 Milvus 或 pgvector）；
3. 所有 memory 写入时同步生成 embedding，注意限制文本长度（超过 512 token 的要做 chunk）；
4. recall 时将 query 做相同 embedding，按 cosine 相似度取 top_k。

**混合检索实操：**
真实 case 里单一方案都无法覆盖。我们在 OpenClaw agent 中实作了 hybrid 逻辑：
- 全文做第一路召回，语义做第二路；
- 两路结果通过 RRF（Reciprocal Rank Fusion）合并；
- 最后加一层 rerank（用 Cross-Encoder 或 LLM 打分），输出最终 top_n。

# 踩坑清单

- **分词器与业务语言不匹配**：医疗、法律等垂直领域用通用词典，全文检索直接失效。解决办法是在 ES 的 jieba 插件中添加自定义字典，或切换到 N-gram 分词。
- **向量检索的相似度阈值陷阱**：不是所有 >0.7 的结果都相关。需要针对 memory 集合校准阈值，通常 0.75~0.82 是中文任务的可用区间，但必须人工评估。
- **多模态 memory**：如果 memory 里包含图片描述、代码块、JSON 输出，单独用文本语义或全文都抓不住结构信息。需要对不同类型字段建独立索引，然后在 recall 层做结果组合。
- **状态记忆 vs 事实记忆不分**：用户状态变化（如“我升级了订阅”）需要强时效性，全文搜索按时间衰减权重比语义搜索更好控，因为语义搜索会把新旧记忆混在一起，容易把过期信息排到前面。
- **成本失控**：语义搜索一旦开启全量 re-embedding，几千条 memory 就会烧掉不少 GPU 或 API 配额。推荐在 memory plugin 中开启增量 embed 策略，并设置 cleanup 机制。

# 可复用的改进建议

1. **优先上 hybrid，但别忘 rerank**：只用 RRF 融合，噪声仍然多。加一个轻量 rerank 层（如 bge-reranker-large）就能把准确率拉高 10~15 个点。
2. **memory 生命周期管理**：对对话历史加 TTL，定期归档或降级冷数据。语义搜索库保持热数据在 2000 条以内，搜索延迟可控。
3. **用户意图改写**：在 recall 前用 LLM 把口语化 query 转成检索友好形式，同时提取关键词。这个步骤对两种搜索都有增益，尤其是将“上次那个事怎么样了”变成“查询上次讨论的需求变更结果”。
4. **落地监控**：记录每一次 recall 的 query、返回结果、用户是否采纳（implicit feedback）。积累一周数据就能评估召回 badcase，针对性地调整分词、阈值或 embedding 模型。

# 总结

语义搜索和全文搜索是 memory_recall 的两条腿，缺一条都会摔倒。在 OpenClaw agent 的实践中，没有一刀切的答案：**需要精确匹配 ID、术语、命令的场景，用全文搜索更可靠；需要理解意图、关联相似表达的对话记忆，用语义搜索更智能。**

工程上更稳妥的路线是先用全文搜索兜底，用最小的向量搜索模块做增量实验，逐步过渡到双路 hybrid。别从第一天就搞复杂的向量库栈，除非你的记忆规模一开始就是 10 万条级别。让 retrieval 服务于 agent 行为一致性，而不是炫技，这是踩完坑后最想强调的一句话。

---

