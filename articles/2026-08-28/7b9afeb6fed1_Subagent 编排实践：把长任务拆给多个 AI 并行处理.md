---
title: Subagent 编排实践：把长任务拆给多个 AI 并行处理
feedId: 35110
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 这类 agent 环境里，单个 agent 处理复杂任务时经常出现几个问题：上下文越滚越长、步骤串行等待、职责混杂导致频繁返工。比如“分析 20 个数据源并生成报告”，如果让一个 agent 从头跑到尾，不仅慢，中间任何一步走偏都会污染后续判断。

subagent 编排的核心思路是：主 agent 只做任务拆分、结果校验和汇总，真正耗时或可独立执行的部分交给多个 subagent 并行完成。这个模式更接近工程里的 fan-out/fan-in，而不是让多个 AI 无目的地“讨论”。

## 先想清楚问题

并行不是简单多开几个 agent。容易翻车的点集中在五个地方：

- 任务边界不清晰，subagent 之间重复劳动或互相等待。
- 上下文复制过多，每个 subagent 都加载完整背景，token 消耗翻倍。
- 工具并发冲突，多个 subagent 同时写同一文件、同一数据库行或同一浏览器会话。
- 输出格式漂移，汇总时主 agent 要花大量精力解析自然语言结果。
- 失败扩散，一个 subagent 返回错误或超时，拖垮整个汇总链路。

## 做法 / 步骤

### 1. 主 agent 只做调度，不干重活

主 agent 的职责压缩成三类：拆任务、校验输出、合并结果。需要长时执行的检索、解析、生成都不放在主 agent 里。这样可以避免主 agent 上下文被细节占满。

### 2. 为每个 subagent 定义小闭环

每个 subagent 必须拿到四个东西：明确目标、最少上下文、可用工具、输入/输出 schema。例如一个“数据源摘要” subagent 只接收 URL 列表和字段要求，输出 JSON：

```json
{
  "source": "url",
  "status": "ok|partial|failed",
  "summary": "...",
  "evidence": ["..."]
}
```

不要给它无关的背景文档。

### 3. 工具通过 MCP 或插件封装成可重入调用

如果多个 subagent 都要查外部服务，最好通过 MCP 暴露成无状态工具。共享可变资源要提前处理：能只读就不给写权限，能写独立路径就不写同一文件，能按 ID 分片就不让两个 agent 碰同一行数据。

### 4. 控制并发，不要一次开满

在 OpenClaw 中实际建议先从 2-3 个并发开始。主 agent 产出一个任务队列，每个任务映射一个 subagent。并发过高时，工具端限流、模型速率限制、上下文切换都会让整体质量下降。用日志里的队列深度和完成时间来判断是否加并发。

### 5. 结果回收要有结构

subagent 输出统一走结构化 JSON 或 Markdown 模板。主 agent 回收时先校验字段是否齐全，不齐全的标记为 `partial`，而不是直接丢弃。对于失败任务，只重试一次，仍失败则进入“人工或降级路径”。

### 6. 汇总时保留证据

主 agent 汇总多个 subagent 的结果时，不要求它凭记忆合并。让它基于带引用或来源的片段去重、去冲突，最后生成结论。保留原始 subagent 输出，方便回溯。

## 踩坑点

### 上下文复制爆炸

一开始容易把整份需求文档塞给每个 subagent。后来改成只给任务切片：目标、输入数据、输出 schema。背景信息由主 agent 预处理后按需分发，token 成本降了接近一半。

### 工具状态冲突

典型场景是两个 subagent 同时操作同一个浏览器 profile 或写同一个临时目录。解决办法是资源分片：A 负责前半段数据，B 负责后半段；或者使用锁/队列，但引入锁会让并行退化成串行，要谨慎。

### 递归调度失控

subagent 有时会自己再开 subagent，层级一深，主 agent 失去控制。实践中限制为“只允许一层 subagent”，即 subagent 不允许再创建下一级 agent。如果需要二次拆分，由主 agent 重新规划。

### 输出漂移

即使给了 schema，部分模型仍会返回一大段自然语言。此时不要让主 agent 强行猜测。用二次解析或简单字段提取，解析失败就标记 `failed`，避免污染汇总。

### 成本放大

并行 N 个 agent 不等于效率提升 N 倍。如果任务拆分不当，多个 subagent 会重复查询相同数据。先跑 1-2 个样本，确认单个 subagent 的步骤数和 token 消耗，再决定是否全量并行。

## 可复用建议

- 所有 subagent 输出带 `run_id`、`task_id`、`status`，日志按 run_id 串联。
- 工具错误映射成结构化错误码，不要让 subagent 自己解释错误。
- 关键路径保留单 agent 版本作为 fallback，subagent 只作为加速手段。
- 给每个 subagent 设置 max steps 和 max tokens，避免单个任务长尾拖慢整体。
- 汇总前先做 schema 校验，不合规结果不进最终报告。

## 总结

subagent 编排适合可拆分、相互独立、结果可结构化的任务，比如批量数据处理、多数据源校验、多文件分析、并行搜索与对比。它不适合强顺序依赖、共享可变状态或需要持续判断的流程。

实际收益来自清晰的边界、受控的并发和严格的回收，而不是“同时开很多 AI”。在 OpenClaw 环境里，结合 MCP 工具和插件，把每个 subagent 当成一个无状态执行单元来设计，通常比让一个巨型 agent 硬扛更稳定，也可控得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9e9ae8a735289fb1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/62b0e836dd90d489.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ffc57b82babcb348.png)

