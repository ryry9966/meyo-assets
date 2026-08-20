---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索，别急着上向量库
feedId: 33914
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw、Agent、MCP 工具链里做 memory_recall，常见两条路线：

- 全文搜索：SQLite FTS5 / BM25，按关键词、ID、命令、路径精确匹配。
- 语义搜索：embedding + 向量相似度，按意图和改写表达召回。

很多同学一上来就默认“语义搜索更高级”，先上向量库。但实际工程里，memory_recall 的目标不是“搜得像搜索引擎”，而是把对当前任务有用的历史片段捞回来。语义搜索在不少场景下会翻车，尤其是精确信息召回。

## 问题

memory_recall 里两类 query 差别很大：

- `OPENCLAW_MCP_PORT`、`/home/user/.openclaw/config.yaml`、`error: timeout after 5000ms` 这类需要精确 token。
- “上次那个 MCP 工具连不上是怎么处理的”“用户不喜欢太啰嗦的回复”这类需要语义理解。

单用全文搜索，同义改写召回很差；单用语义搜索，专有名词、代码符号、版本号、路径容易混。真正可用的 memory_recall，通常不是二选一，而是先有全文基线，再用语义兜底，最后混合召回。

## 做法 / 步骤

**1. 先定义 recall 单元**

不要拿原始日志当记忆。建议按 session 摘要、工具调用对、用户偏好、任务结论等结构化 chunk 写入。每个 chunk 控制在 200–600 token，或按 JSON 块切分，避免语义被稀释。

**2. 先做全文搜索基线**

用 SQLite FTS5 对 `content`、`namespace`、`tags` 建虚拟表，写入 memory 时同步插入。先不接任何向量库，跑 20–30 条固定 query，看命中情况。

FTS5 对中文默认分词不理想，可以用 `tokenize='trigram'`，或者在写入前用 jieba 等做预处理。OpenClaw 场景里很多英文命令和路径，FTS5 对这类内容表现稳定。

**3. 再加语义索引**

用本地 embedding 模型（如 bge-m3 或 OpenAI 小模型）对同一 chunk 生成向量，存到 sqlite-vec、chroma 或 pgvector。阈值先设 0.35–0.5，不要默认 0.7。向量相似度阈值不可跨模型迁移，换了 embedding 模型需要重评。

**4. 混合召回**

两种召回各自取 top_k，再用 RRF 或加权合并。建议先用 RRF，k=60 起步。不要一上来就细调权重，固定 0.5/0.5 或 RRF 已经能覆盖大部分场景。

**5. 建立 eval set**

每条 eval 包含：query、相关 memory id、干扰项。指标看 recall@5 和 MRR，不要只看“有没有命中一条”。eval 里必须包含四类 query：

- 精确 ID / 命令 / 路径
- 同义改写
- 跨 session 关联
- 时间敏感信息

**6. 记录线上结果**

把每次 recall 的结果、最终被 agent 采用哪条、用户是否纠正写进日志。周级抽查，比单纯调模型更有效。

## 踩坑点

- **语义搜索对代码符号不敏感**：`OPENCLAW_MCP_PORT` 和 `openclaw mcp port` 可能被 embedding 拉得很近，但实际不是一个东西。这类字段必须走 FTS。
- **中文全文搜索别用默认分词**：FTS5 的 unicode61 对中文按连续字切，召回质量会很差。要么 trigram，要么写入前预分词。
- **向量阈值别照搬教程**：不同 embedding 模型、不同 chunk 策略，相似度分布完全不一样。0.7 对某些模型根本召不到。
- **增量更新容易漏 embedding**：memory 写入时只更新了 FTS，忘了同步向量索引。建议写入逻辑统一走一个接口，不要两处调用。
- **chunk 太大导致语义被稀释**：一段 2000 token 的原始转录里包含十几个信息点，语义搜索很难定位到具体那一句。
- **混合融合前期别调参**：先跑通 RRF，再根据 eval 微调。否则很容易在 20 条测试集上过拟合。

## 可复用建议

- **先上全文搜索**。OpenClaw / Agent / MCP 场景下，大量 recall 需求是精确的：工具名、参数、报错、路径。FTS5 足够快，也足够稳。
- **语义搜索只做兜底**。它适合“用户换了个说法”“意图相似但措辞不同”的场景，不要让它承担精确匹配。
- **用 namespace + tag 先过滤**。比如按项目、用户、工具名隔离，减少搜索空间，比提升模型效果更直接。
- **eval set 要和真实日志对齐**。从线上日志里抽 query，不要人工编理想 query。人工 query 通常太干净。
- **别上重型向量库**。如果 memory 量在万级以内，SQLite FTS5 + sqlite-vec + 本地 embedding 完全够用，部署和排障成本低很多。

## 总结

memory_recall 没有“语义搜索一定比全文搜索好”这种结论。工程上务实的路线是：

1. 先做全文搜索，覆盖精确信息；
2. 再用语义搜索兜底改写和模糊意图；
3. 用 RRF 混合召回；
4. 用固定 eval set 和线上日志评估，不靠感觉调参。

在 OpenClaw 这类 Agent 系统里，稳定可解释的召回，比追求“更智能”的向量检索重要得多。

---

