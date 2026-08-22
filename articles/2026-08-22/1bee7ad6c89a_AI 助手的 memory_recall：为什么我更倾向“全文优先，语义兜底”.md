---
title: AI 助手的 memory_recall：为什么我更倾向“全文优先，语义兜底”
feedId: 34235
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 这类长期运行的 Agent 里，memory_recall 是决定“上下文里塞什么”的关键一环。现在大家习惯把所有记忆向量化，用 embedding 做语义搜索，但实际在命令、配置、报错、ID 这类记忆上，语义搜索常常给出看似相关、实则不能用的结果。本文记录我在插件和自动化任务里对 memory_recall 的选型与混合检索实践。

## 问题

记忆库里存的往往不是文章，而是高度稀疏的操作痕迹：shell 命令、API 路径、错误码、用户偏好短语、MCP 工具名。语义搜索会检索“意思相近”的条目，但 Agent 需要的是“准确匹配”。

例如 query 是 `pg_dump 超时`，全文搜索能直接命中包含 `pg_dump timeout` 的记忆，而语义搜索可能返回一堆 PostgreSQL 故障贴，却漏掉确切的命令。反过来，用户说“把那个清理脚本再跑一次”，语义搜索能懂“清理脚本”≈“cleanup.sh”，全文搜索不懂同义改写。单一检索方式都要吃亏。

## 做法/步骤

1. **先定义记忆 schema**  
   `memory_id, content, type(command/config/summary/preference), tags, created_at, last_accessed_at, importance`

2. **建全文索引**  
   对 `content`、`tags`、`type` 建 SQLite FTS5 表。中文用 trigram 或 jieba 预切分。对 command 类记忆，存储规范化后的小写字符串，例如 `pg_dump timeout`。

3. **建语义索引**  
   只对 summary/preference 等自然语言内容做 embedding，存 Chroma 或 pgvector。短命令、ID、版本号不必向量化。

4. **memory_recall 内部做混合检索**  
   - 第一阶段：精确全文检索，`top_k=5`，BM25 打分  
   - 第二阶段：语义检索，`top_k=10`，相似度阈值 0.35–0.5  
   - 合并：RRF score = `sum(1/(k + rank))`，k 取 60  
   - 合并后按元数据过滤：只保留最近 30 天或 `importance > 0` 的条目，并控制总 token < 1500

5. **返回时标记来源**  
   `"source": "fts"` 或 `"source": "vector"`，让上层 LLM 优先采信 fts 命中；如果 fts 无结果，才推荐 vector。

6. **更新同步**  
   写入 memory 时同时写 FTS 表和向量库，异步任务做一致性检查。

## 踩坑点

- **中文全文分词**  
  默认 FTS5 按字符切，英文没问题，中文搜“清理脚本”可能拆成单个字，召回太多。用 trigram tokenizer 或 jieba 预切分后存空格分隔。

- **向量阈值别定太高**  
  语义相似度 0.7 可能一个都召回不到，0.3 又太多噪声。建议根据 embedding 模型在本地评测集上调。

- **RRF k 值**  
  k 越小越强调靠前排名，k 越大越平滑。我固定 60 是经验值，不要随便调。

- **索引不同步**  
  记忆更新后旧向量仍在，导致召回过期内容。写入时用 `memory_id` 覆盖，删除时显式调用 delete。

- **Token 超限**  
  memory_recall 返回太多候选，把系统提示撑爆。必须在合并后做 token 预算截断，而不是在检索前。

- **语义搜索对版本号、IP、命令参数不敏感**  
  `python3.11` 和 `python3.10` 在向量空间可能很近，但实际不可互换。这类字段不要依赖向量。

## 可复用建议

- 小规模记忆（< 5000 条）别上 Elasticsearch，SQLite FTS5 + Chroma 就够，部署简单，Agent 走本地调用延迟低。
- 先上全文，后上语义。如果记忆类型以操作型为主，全文召回率已经能覆盖 80% 的需求。
- 为 memory_recall 增加一个 `search_type` 参数：`hybrid|fts|vector|auto`，让调用方按场景选择。命令补全用 `fts`，偏好问答用 `vector`，默认 `hybrid`。
- 记录查询日志：`query, recall_ids, clicked_id`。跑一周后离线分析，能看出哪些 query 该命中但没命中，针对性调整。
- 如果 memory_recall 是 MCP 工具，把混合检索封装在 server 端，外部无需关心实现。

## 总结

语义搜索不是 memory_recall 的默认解，全文搜索也不是过时技术。对 Agent 自动化来说，可复现、可命中的精确检索优先，语义检索负责补“同义改写”的盲区。工程上我建议：全文索引做第一道召回，向量检索做第二道，RRF 合并，再按元数据和 token 预算截断。先保证“找到”，再追求“找巧”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9000f71979aeb21e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/380c7a04ea448cce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c677242668bdedab.png)

