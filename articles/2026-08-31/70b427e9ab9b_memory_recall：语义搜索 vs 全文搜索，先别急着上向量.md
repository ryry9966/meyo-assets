---
title: memory_recall：语义搜索 vs 全文搜索，先别急着上向量
feedId: 35555
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw/Agent/MCP 的自动化链路里，memory_recall 通常负责把历史记忆、日志片段、skill 说明捞出来塞进上下文。随着 memory 膨胀，检索质量直接决定 Agent 是否“记性差”。很多人第一反应是上 embedding + 向量库，觉得语义搜索更聪明；但在工程上，这不一定划算。

## 问题

全文搜索和语义搜索的差异不是“模糊 vs 精确”，而是匹配逻辑不同。

- 全文搜索：对错误码、路径、命令、函数名、专有名词敏感，但怕同义改写、跨语言和中文分词。
- 语义搜索：能理解“容器一直重启”和“pod restart loop”是同一类问题，但对 `EAI_AGAIN`、`pod-123`、`/var/log/foo` 这种精确串不敏感，还可能被主题相似但无关的条目带偏。

如果你的 memory 里大量是日志和排障记录，语义搜索可能不如预期。

## 做法/步骤

1. 先盘点 memory 类型：结构事实、日志片段、代码、总结。精确字段有多少，模糊描述有多少。
2. 先建全文索引，推荐 SQLite FTS5，OpenClaw 插件容易接。中文不要用默认 unicode61，用 trigram 或 jieba 预分词。
3. 准备 30-50 条真实查询，标记每条查询应该命中的条目。跑 recall@5，别只看单条。
4. 如果全文 recall@5 低于预期，再上语义索引。本地 embedding 用 bge-small-zh 这类即可，chunk 控制在 256-512 token。
5. 两条索引都建好后，用 RRF 融合，而不是直接加权。`score = Σ 1/(60 + rank_i)`，对两组排序做合并，避免 BM25 分数和余弦相似度量纲不一样。
6. 上线后把未命中查询落盘，每周看漏召回原因。

可参考配置：

```yaml
memory_recall:
  primary: fts5
  secondary: vector
  fusion: rrf
  exact_fields: [error_code, command, path, api_name]
  top_k: 8
  min_score: 0.35
```

## 踩坑点

- 直接加权融合容易烂。BM25 分数范围不稳定，余弦相似度又集中，必须归一化或用 RRF。
- 中文 FTS5 默认分词对中文极差，会把整句当 token，导致召回很低。
- 对错误码、异常类、commit hash 只做语义索引，会漏原始日志。这些字段应走 exact match 或全文保护。
- chunk 切太碎或太长。太碎丢失上下文，太长向量被稀释。优先按 memory 条目的逻辑边界切，不按固定字数硬切。
- 编辑 memory 后向量索引不更新，旧数据反复被召回。需要重建或增量同步。
- 不要只调 top_k 掩盖质量差。先看 score 分布和 min_score，避免大量弱相关条目占位。

## 可复用建议

- 默认“全文为主、语义为辅”，不要一上来 all-in 向量库。
- 可精确匹配的字段单独提取，查询命中时直接加权。
- 中文 memory 用 trigram，英文 memory 开 stemmer。
- 评估看 recall@5 和 MRR，不只凭感觉。
- memory 总量几千条以内，FTS5 完全够用；真正需要语义搜索的是大量非结构化、长文本、跨表达场景。
- 把 embedding 模型和检索器解耦，方便换模型或下线语义索引。

## 总结

memory_recall 的选型不是“语义搜索 vs 全文搜索哪个更好”，而是先明确你的记忆里有多少精确字段、多少模糊语义。工程上先做全文基线，用数据决定是否加语义，最后用 RRF 做混合。这样排障路径清晰，也避免为“看起来智能”支付不必要的复杂度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2c3ad17f099110a7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/3c542cb78f3df5b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/cb4c088d951c4023.png)

