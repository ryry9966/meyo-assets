---
title: AI 助手 memory_recall 选型：语义搜索与全文搜索的工程对比
feedId: 34048
source: 综合讨论
publishedAt: 2026-08-21
---

# AI 助手 memory_recall 选型：语义搜索与全文搜索的工程对比

## 背景
在 OpenClaw agent 里，memory_recall 通常不是“让模型自己回忆”，而是由一个 MCP 工具或内部模块先检索，再把候选记忆塞进上下文。这个检索层选型不当，上层 prompt 写得再好也会被错误记忆带偏。

我们最早给 agent 接 memory MCP 时，只挂了 embedding 向量检索。查“上次部署报错 EACCES 怎么处理的”，返回的是几条“权限问题”相关但不完全对的历史；查“用户偏好回复风格简洁”，又经常漏掉“回答要短”这种同义表达。后来补了一条全文检索链路，问题才收敛。

## 问题
两种路线各有擅长：

- 语义搜索：对同义改写、自然语言意图友好，但精确 token（错误码、路径、命令、日期）不可靠。
- 全文搜索：对精确 token 友好，但对“短一点/简洁/别啰嗦”这类改写不敏感。

所以重点不是选哪个，而是“你的记忆数据长什么样”。如果记忆里大量是日志、报错、命令历史，全文必须存在；如果是偏好、用户事实、总结性内容，语义更重要。

## 最小可复现做法
**1. 准备样本**
从真实使用里抽 200 条记忆，分成四类：偏好/事实、任务日志、报错堆栈、代码片段。再标注 20 个查询的 ground truth，用 recall@5 评估，不要凭感觉。

**2. 全文线**
SQLite FTS5 足够起步。注意中文不要用默认 unicode61，它不会按词切分。用 trigram tokenizer：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  tokenize='trigram'
);
```

查询：

```sql
SELECT content
FROM memory_fts
WHERE memory_fts MATCH ?
ORDER BY bm25(memory_fts)
LIMIT 5;
```

如果内容较长或需要更准分词，可以先 jieba 分词，写入 shadow column 再建 FTS。

**3. 语义线**
本地 embedding 模型或接口 embedding 都可。chunk 控制在 300~500 token，块间重叠 50 token。用 cosine 检索 top K。OpenClaw 侧可以用 memory MCP server 暴露一个 `memory_search(query, mode)` 工具，让 agent 决定检索模式。

**4. 混合排序**
不要直接相加 embedding score 和 bm25 score，量纲不同。用 RRF 更稳：

```
RRF(rank) = Σ 1/(60 + rank_i)
```

通常 embedding 和全文各取 top 20 融合，再取前 5。

## 踩坑点
- **中文全文别裸奔**：默认分词会把整句当一个 token，recall 接近零。trigram 是最小可用方案，但注意会引入噪声。
- **语义搜索对专有名词不敏感**：`EACCES`、`v2.3.1`、`/data/backup` 这类 token 的 embedding 相似度可能不如“权限”高，导致召回不精确。报错码、版本号、路径应保留原文，并给全文线路更高权重。
- **相似度阈值不要写死**：0.7 在某个中文 embedding 上可能漏掉大量正确项。按你自己的验证集标定，或者干脆不设硬阈值，只返回 top K 和 score，让上层决定。
- **更新链路**：记忆更新时，DB 和向量容易不一致。MVP 阶段可以用同一事务写结构化字段，向量异步重建；规模大了再考虑定期全量重建。
- **长文档检索**：chunk 太小会切碎错误堆栈，太大会把正确段落淹没在相似度里。300~500 token 是工程起点，不是绝对标准。

## 可复用建议
1. **默认“全文兜底 + 语义扩展”**：先 FTS 精确召回，再用语义补同义改写。这样报错码和自然语言查询都不会太差。
2. **按记忆类型分层**：长期偏好、用户事实走语义；近期操作、命令历史、报错日志走全文，并加时间衰减。近期的精确错误码往往比旧的偏好更重要。
3. **MCP tool 设计上给 mode**：`memory_search(query, mode='hybrid', limit=5)`，默认 hybrid，允许 agent 在明确知道要查错误码时切 `fulltext`。
4. **记录召回效果**：每次召回后记录 query、返回候选、最终是否被采纳。跑几轮就能看出哪类 query 应该调权重或调 chunk。
5. **不必一上来上向量数据库**：SQLite FTS5 + 本地 embedding + RRF，足以覆盖大多数个人 agent 和小团队场景。

## 总结
语义搜索和全文搜索不是替代关系。memory_recall 如果只抱着 embedding，会在错误码、路径、命令历史上翻车；如果只抱着 FTS，又会在同义改写和偏好查询上漏召回。工程上更划算的是混合检索、分层记忆、可评估的最小闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/75e21cc48e37f3cb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b58282d8ee449768.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/eb4b80ca774e535c.png)

