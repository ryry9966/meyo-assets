---
title: 语义搜索不是替代品：AI 助手 memory_recall 的工程化选型
feedId: 34656
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent 项目里，memory_recall 很容易被当成“接上 embedding 就能用”的功能。尤其是做插件或 MCP memory server 时，第一版通常只接一个向量搜索，跑通 demo 就上线。等记忆条目从几百条涨到几万条，问题才开始暴露：助手答非所问、漏掉关键上下文、重复询问用户已经说过的偏好。

对工程实践者来说，核心问题不是“要不要语义搜索”，而是：**语义搜索和全文搜索到底该选哪个，什么时候选，怎么组合。**

## 问题

语义搜索擅长处理同义改写、模糊意图、跨语言联想。比如用户说“上次那个跑不起来的服务”，它能找到“API 启动超时”的记忆。全文搜索擅长精确匹配：命令、路径、ID、报错码、代码片段。例如 `/opt/scripts/backup.sh` 或 `ERR_SSL_PROTOCOL_ERROR`，语义向量往往无力区分，甚至会把不相关内容拉进来。

单用语义搜索，数字、路径、变量名会被模型“平滑掉”，精确事实反而模糊。单用全文搜索，用户换个说法、用近义词或口语化描述，就搜不到。工程上没有银弹，只有按召回目标做分流的混合检索。

## 做法/步骤

1. **先定义 recall 目标。** 把 memory 拆成可过滤的元数据字段：`type`、`source`、`created_at`、`conversation_id`、`importance`。不要把元数据塞进语义向量，它们应该走结构化过滤。

2. **全文通道用 SQLite FTS5 或 Postgres FTS。** 字段只放需要精确匹配的内容：`title`、`keywords`、`commands`、`error_code`、`raw_snippet`。中文场景注意 FTS5 默认 tokenizer 对中文分词不友好，可以用 trigram 或 jieba 预分词后写入。

3. **语义通道用 embedding + 向量索引。** 小规模用 sqlite-vec 或 Chroma，大规模用 pgvector 或 Qdrant。只对“语义内容”建向量，例如 `summary`、`preferences`、`lessons_learned`。向量查询必须限制 `top_k`，并设置相似度阈值。

4. **查询入口做分流。** 先做轻量 query 分类：如果包含路径、命令、报错码、ID，优先走全文；如果更像自然语言描述，走语义。不要上复杂意图模型，规则加关键词特征即可。

5. **合并结果用 RRF 或简单加权。** 最后做一次轻量 rerank：把候选记忆和当前 query 拼在一起，用本地小模型或规则打分。rerank 不是必选，但能明显降低“语义搜索前几名不相关”的问题。

6. **建立评测集。** 准备 50-100 条真实 query，标记应召回的记忆 ID。每次换 embedding 模型、改阈值、改融合权重，都跑同一份评测集。记录 `recall@5`、MRR 和误召回率。

## 踩坑点

- **embedding 模型切换会导致旧向量不可比。** 必须做模型版本号，改模型后全量重建索引。
- **语义搜索对短文本、专有名词、数字不敏感。** 不要把 UUID、端口号、日期范围只放在向量里。
- **全文搜索的分词和大小写处理要可复现。** 英文可以统一 lowercase，但命令和路径不要 lower；中文预分词要固定脚本。
- **相似度阈值不能拍脑袋。** 不同 embedding 模型的分数分布差异很大，必须用评测集标定。
- **召回条数不是越多越好。** 把 top 20 塞进上下文，可能把真正答案挤出窗口，还增加 token 成本。
- **忽略元数据过滤。** 只靠向量搜索，时间一久会把过时、低优先级的记忆反复召回。

## 可复用建议

- 默认“全文兜底，语义增强”。全量上线前先跑一周 shadow mode，只记录召回结果、不注入上下文。
- 给 recall 返回结果带上 `score`、`source`、`type`、`updated_at`，调试日志里要能看出为什么召回。
- 把用户纠正和漏召回 case 沉淀成评测集，比反复调参更有效。
- 小项目不要一上来就上 vector DB。SQLite FTS5 + 一个本地 embedding 文件就能覆盖几千条记忆。
- 如果是 MCP memory server，把 recall 工具设计成可传 `filter` 参数，让 Agent 自己决定 type 和时间范围，不要让模型盲目调用无过滤召回。

## 总结

memory_recall 不是单一检索算法问题，而是召回策略、索引结构、评测和 fallback 的组合。语义搜索适合模糊意图，全文搜索适合精确事实。真正能稳定工作的做法是：**元数据过滤 + 双通道召回 + 阈值标定 + 可观测 trace**。与其追求“更好的 embedding”，不如先把评测集和分流规则建起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/cb3e225c79639687.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/75f2be72528b3ddc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c67c04948ea48478.png)

