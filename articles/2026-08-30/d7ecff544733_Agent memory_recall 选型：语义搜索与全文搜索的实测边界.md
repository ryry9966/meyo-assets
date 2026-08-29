---
title: Agent memory_recall 选型：语义搜索与全文搜索的实测边界
feedId: 35273
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 的 Agent 插件里，memory_recall 不是独立 RAG 项目，而是服务任务执行：每次工具调用前把相关上下文塞进 prompt，或者召回上轮偏好。实现上，很多人会直接上 embedding + 向量库，默认语义搜索一定比全文搜索好。我们在自建 memory_recall 插件时做了小规模对照，结论不太一样。

## 问题

实际 memory 数据大概三类：

- 命令、报错、日志片段：如 `EACCES: permission denied`、`/opt/openclaw/bin/start.sh`。
- 半结构化偏好：如“用户希望所有写入任务先输出 dry-run”。
- 长会话语义：“上次调优时把 timeout 从 3s 改成 10s，因为 CI 机器慢”。

全文搜索擅长第一类，语义搜索擅长第三类；第二类两者都可能命中。问题不是哪个更好，而是你的 memory 数据更偏哪一类，以及检索失败时怎么兜底。

## 做法/步骤

我们用 SQLite FTS5 做全文检索，用 sqlite-vec 存本地 embedding。流程：

1. 建表：`memory(id, session_id, content, meta, created_at)`，内容按段落或 500 字切块。
2. 建全文索引：使用 SQLite FTS5 trigram tokenizer，中文不要用默认 unicode61：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  tokenize = 'trigram'
);
```

3. 建向量索引：本地 embedding 模型用 `paraphrase-multilingual-MiniLM-L12-v2`，把每块编码成 384 维，归一化后写入 vec0 表。
4. 写统一召回函数：全文用 `bm25(memory_fts)` 排序，语义用 cosine 相似度；先取 top20，再用 RRF 合并：

```text
rrf_score = 1/(60 + rank_fts) + 1/(60 + rank_vec)
```

5. 用 46 个历史查询做评测，指标看 recall@5：至少一个相关 chunk 出现在前 5 个结果里的比例。

结果：纯全文 recall@5 0.71，纯语义 0.78，RRF 混合 0.89。差距主要来自报错码/路径类和同义改写类。前半类全文赢，后半类语义赢。

## 踩坑点

- **FTS5 默认分词对中文不友好**：`permission denied` 能查，`权限被拒绝` 查不到词根。上 trigram 后召回上升，但索引体积变大。
- **embedding 模型别用纯英文小模型**：短句“日志出现 timeout”和“超时错误”完全同义，英文小模型可能给出很低相似度；换多语言模型有明显提升。
- **embedding 更新负担比想象大**：只要 chunk 规则、清洗脚本、模型版本发生变化，历史向量都得重算。所以先固定 chunk 规则再评测，不要边调边加数据。
- **全文对符号和 ID 很强，语义很弱**：`k8s_worker-7` 这类标识符 embedding 编码后容易和普通词混在一起，语义检索会漏；但 FTS5 一下就能命中。
- **短 document 归一化后 cosine 不稳定**：连续的“存活 / 已停止 / 重启完成”三个状态很容易互相高相似，必须加 `session_id`/`task_type` 过滤。

## 可复用建议

1. 先做 20-50 条真实查询的 golden set，再决定上不上向量库，不要凭感觉。
2. 日志、命令、路径、ID 密集的 memory，优先 FTS5 全文，并保留 `LIKE`/正则兜底。
3. 偏好、会议纪要、错误解释这类同义改写多的，优先语义。
4. 最后才考虑混合检索：语义分和全文分不要直接相加，用 RRF 或最小-最大归一化；否则长文本全文分数会压过语义。
5. 给 memory_recall 增加 `--search-mode auto`：先做一次全文，若结果覆盖不到阈值再补语义，成本更低，效果接近混合。
6. 检索前先过滤：`WHERE session_id = ? AND task_type = ?`，比所有模型技巧都更有效。

## 总结

memory_recall 的语义搜索 vs 全文搜索不是替代关系，是数据形态问题。工程上更值得做的是：固定评测集、先加过滤条件、用 FTS5 trigram 接住精确词，用本地多语言 embedding 接住语义改写，最后用 RRF 合并。这样在 OpenClaw 插件里跑得更稳，也不会被向量库的复杂度绑架。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7089d28837972119.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ea715c1b69b20691.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/271a3a08dbaaffc1.png)

