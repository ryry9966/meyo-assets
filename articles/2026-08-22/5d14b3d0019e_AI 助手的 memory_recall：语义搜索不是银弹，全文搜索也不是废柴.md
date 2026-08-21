---
title: AI 助手的 memory_recall：语义搜索不是银弹，全文搜索也不是废柴
feedId: 34095
source: 综合讨论
publishedAt: 2026-08-22
---

### 背景
OpenClaw 的 memory_recall 是 Agent 能否“记住”历史的关键，但很多实现只做一件事：要么用 SQLite FTS5 全文搜索，要么直接上 embedding 语义搜索。我在给 OpenClaw 做 memory 插件时，经历了从纯 FTS5 到纯语义，再回到混合检索的过程。这里记录可复现的做法和坑。

### 问题
全文搜索（FTS5）精准但脆弱。比如记录里有“CF 524 timeout”，用户问“Cloudflare 边缘网络 5xx 怎么修的”，关键词匹配不到。语义搜索能解决这类改写，但反过来会把语义相近但无关的内容召回，比如问“数据库连接池超时”，它可能给你“HTTP 连接池大小调整”。

所以问题不是选哪个，而是 memory_recall 需要同时具备“词面匹配”和“语义扩展”，并且有过滤和重排兜底。

### 做法/步骤
1. 全文索引不要拆掉  
SQLite 建表时用 FTS5 做为主索引：
```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content, source, tokenize='unicode61'
);
```
写入 memory 时同步更新 FTS。中文场景建议换成 `tokenize='trigram'`，因为 unicode61 对中文分词不友好，容易整段成一个 token。

2. 语义索引按需引入  
记忆条数小于 5000 条，可以不做向量索引，每次 recall 直接对过滤后的候选全量算余弦相似度。条数再多，再用 sqlite-vec 或外部向量库。  
embedding 模型选择本地小模型即可，例如 bge-small-zh 或 all-MiniLM-L6-v2。固定模型版本，记录维度。

3. 召回与合并  
FTS5 取 top20，语义取 top20，用 RRF 合并：
```text
rrf_score = sum(1 / (rank + 60))
```
也可以加权合并，但 RRF 不依赖分值分布，更稳。

4. 加一层 LLM 重排  
混合召回后的候选不要直接返回。把 query 和每个记忆的前 200-300 字交给 LLM，输出 0-1 的相关分。这一步成本低，效果提升明显。

5. 固定评测集  
准备 30-50 条真实历史问题，标注应命中的 memory id。每次调整权重或换 embedding 模型，跑 recall@5 和 MRR。

### 踩坑点
- 中文分词：FTS5 unicode61 默认会把连续中文当整个 token，“边缘网络”和“边缘 网络”召回不到。要么用 trigram，要么写入前做 2-gram 分词。
- 纯语义过度联想：领域术语区分度不够时，语义搜索会给错相同上下文的近似词高分。保留 FTS5 的词面匹配可以压住这类错误。
- 向量更新滞后：memory 内容更新后，embedding 列若异步生成，会漏掉最新记忆。写入路径要同步 upsert embedding。
- 长文本整段向量化：完整日志或会话总结压成一个向量，容易“高分定位不到关键句”。按 200-500 字切片，关联同一 memory id。
- 忽略过滤条件：recall 时应当支持 source、时间范围过滤。先过滤再检索，既快又能减少噪声。

### 可复用建议
- 先上 FTS5，别一上来就重向量。它维护成本低，能覆盖 60%-70% 的 memory recall。
- 语义搜索作为补充，不是替换。只有在模糊改写、多语言、跨记录推理频繁出现时才引入。
- 权重和模型不要频繁换，换 embedding 模型必须重新生成所有向量。
- memory_recall 返回时带上 score 和 source，方便上层 Agent 判断可信度。

### 总结
在 OpenClaw 的 memory_recall 里，比较稳的组合是：FTS5 做词面召回，embedding 做近义补充，RRF 合并，LLM 重排，最后用固定评测集验证。语义搜索能解决很多 FTS 召回不到的问题，但没有词面检索和过滤的纯语义召回，很容易在工程里翻车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/454695557238997a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b9951937aa674510.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a0438040f7dffc22.png)

