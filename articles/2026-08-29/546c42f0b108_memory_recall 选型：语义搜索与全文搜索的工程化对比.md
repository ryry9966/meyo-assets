---
title: memory_recall 选型：语义搜索与全文搜索的工程化对比
feedId: 35133
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw/Agent/MCP/自动化实践里，memory_recall 通常不是“记住一句话”，而是给后续任务提供可召回上下文：工具结果、用户偏好、历史决策、报错片段、接口返回。检索质量直接影响 Agent 是否重复踩坑、是否把无关记忆塞进 prompt。

常见两条路线：

- 全文搜索：BM25、SQLite FTS5、Elasticsearch，依赖词项匹配。
- 语义搜索：embedding + 向量检索，依赖语义相似度。

两者都不万能。本文记录一次把 memory_recall 从“纯全文”切到“混合召回”的过程与判断。

## 问题

早期 memory 表用 SQLite FTS5，存原始文本、时间、来源。查询时直接 MATCH 关键词。优点是快、可解释，缺点是：

- 中英文混合时，分词不一致。
- 用户说“上次那个超时的接口”，记忆里写的是“GET /api/v1/orders timeout”，BM25 找不到。
- 错误码、路径、命令等精确 token 被 FTS 分词后，召回变差。
- 语义相近但用词不同，完全漏召回。

换成全向量库后，也出现新问题：

- embedding 对短文本、命令、ID 不敏感：“ERR_SSL_PROTOCOL_ERROR”和“TLS handshake failed”语义接近但故障不同。
- 新写入记忆后，索引更新不及时，旧向量仍在召回。
- 相似度阈值难调：0.75 太多噪声，0.85 又漏掉关键上下文。

结论：memory_recall 不适合单一路线，应该按内容类型分路，再做混合排序。

## 做法/步骤

1. 明确记忆条目结构

```
{
  "id": "mem_01",
  "type": "tool_result | preference | incident | note",
  "text": "原始内容",
  "keywords": ["timeout", "orders"],
  "embedding": null,
  "created_at": 1710000000,
  "source": "mcp:http_client"
}
```

保留原始 text，不只用 embedding 或分词后的倒排。

2. 全文索引做精确/词项召回

SQLite FTS5 建表时保留原字段，并对命令、路径、错误码额外存一份 `keywords`。查询时：

```sql
SELECT id, bm25(memory_fts) AS score
FROM memory_fts
WHERE memory_fts MATCH ?
ORDER BY score
LIMIT 20;
```

对 `type = 'incident'` 和包含路径/错误码的查询，优先走全文。

3. 语义索引只做召回，不做最终排序

用本地 embedding 模型写入向量库。查询时先召回 top 40，而不是 top 5：

```python
vec = embed(query)
hits = vector_index.search(vec, limit=40)
```

召回多了没关系，后面统一排序。

4. 混合排序

把全文命中和语义命中合并，用 RRF（Reciprocal Rank Fusion）或简单加权：

```python
score = 0.6 * norm(bm25_score) + 0.4 * norm(cosine_sim)
```

对于 `keywords` 精确命中，直接加权 1.5，避免命令、ID 被语义分拉低。

5. 写入路径统一

记忆写入时同步更新 FTS 和向量索引。不要等后台任务异步刷新，否则 Agent 刚写入的决策下一次 recall 不到。可以接受批量写库 + 提交后立刻重建单条倒排，向量写入可异步但要有版本号。

6. 评测集

至少准备 30 条查询，覆盖：同义改写、错误码精确匹配、路径、中文口语、跨工具结果。记录 recall@5 / MRR。没有评测集时，先收集线上未命中 case。

## 踩坑点

- **FTS5 中文分词**：SQLite 默认 unicode61 对中文不友好。可以用 trigram tokenizer，或者写 keywords 字段时按字符 n-gram 拆分。
- **embedding 维度不固定**：换模型后旧向量维度不匹配。向量表加 `model` 和 `dim` 字段，查询时只检索同模型。
- **相似度不是置信度**：0.8 的 cosine 不一定比 0.7 更相关，尤其短文本。别只看分数，要看 type 和时间衰减。
- **全文命中过少**：用 MATCH 时 FTS5 对太短关键词可能返回空。可以降级成 LIKE，但只对 keywords 字段。
- **向量搜索有“新记忆冷启动”**：刚插入的向量如果没有落盘或索引未更新，召回不到。检索前检查 `last_indexed_at`。
- **混杂长文本**：如果 memory 存了完整日志或网页，embedding 会被噪声主导。写入前先截断或摘要。

## 可复用建议

- 默认全文召回，语义仅作为补充。全文能精确匹配的，不要用语义硬凑。
- 短命令、路径、错误码、用户名、工具名：永远保留原始字符串，并给全文索引。
- 语义搜索适合：用户偏好、自然语言描述、同义改写、跨条目的经验归纳。
- 混合排序比单纯切换更稳，RRF 比线性加权更容易调。
- 将 `memory_recall` 输出限制为“短上下文卡片”，而不是原样塞进大段日志。
- 每周统计召回失败 case，补充 keywords 或规则，而不是频繁换 embedding 模型。

## 总结

memory_recall 不是“语义搜索一定比全文好”，也不是反过来。工程上，全文解决精确匹配和可解释性，语义解决同义改写和模糊召回。把两者做成可切换、可混合的检索管道，比纠结单一方案更实际。

对 OpenClaw/Agent/MCP 用户来说，关键是保留原始文本、控制写入链路、限定检索范围、用评测集验证。这样 memory_recall 才能稳定给 Agent 提供上下文，而不是变成“时灵时不灵”的黑盒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f7daa154267b0c4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e849f8fffd6bd4ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/5b073c7245f0cc03.png)

