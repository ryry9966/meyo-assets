---
title: Agent 的 subagent 编排：多个 AI 并行做事的工程实践
feedId: 35630
source: 综合讨论
publishedAt: 2026-09-01
---

# Agent 的 subagent 编排：多个 AI 并行做事的工程实践

## 背景

单个 Agent 处理多个独立任务时，串行执行通常很慢。比如同时要完成“代码仓库结构扫描”“issue 分类”“文档检索”三件事，如果在一个主 Agent 里逐个做，不仅耗时长，而且每步的工具输出都会塞进上下文，很容易撞到 token 上限。

更稳的做法是：把任务拆给多个 subagent 并行处理，主 Agent 只做编排、汇总和异常处理。这个思路在 OpenClaw/Agent 类项目里尤其适用，因为很多时候瓶颈不在推理，而在工具调用和等待。

## 问题

并行不是简单多开几个 Agent。真正要解决的问题包括：

- 任务怎么拆，避免互相依赖；
- 上下文怎么隔离，避免 token 爆炸；
- 结果怎么合并，避免关键信息丢失；
- 失败怎么隔离，避免一个 subagent 拖垮整条流水线；
- 工具权限怎么收窄，避免越权操作。

## 做法

### 1. 先拆任务，不拆执行步骤

按“输入/输出边界”拆，而不是按流程顺序拆。适合并行的任务通常是：

- 独立检索/读取类任务；
- 独立代码审查或文件分析；
- 对同一批数据做不同维度的分类或摘要。

如果任务之间有依赖，先串行主干，再把独立叶子节点并行化。

### 2. 定义 subagent 规格

每个 subagent 都应该有明确的 role、输入 schema、输出 schema、工具白名单、超时时间和最大 token 数。示例：

```yaml
subagents:
  - name: repo_research
    role: 扫描仓库结构与依赖
    tools: [read_file, grep, list_dir]
    output: {type: json, schema: "repo_summary"}
    timeout: 120s
    max_tokens: 3000
  - name: issue_triage
    role: 分类 issue 列表
    tools: [github_list_issues, read_issue]
    output: {type: json, schema: "triage_list"}
    timeout: 90s
    max_tokens: 2500
```

工具白名单很重要。不要默认继承主 Agent 的所有工具，否则一个只做读取的 subagent 可能误触写入或发布操作。

### 3. 限制并发与超时

并发数从 2 开始，不要一上来就开 8 个。很多 MCP server 或本地工具服务有连接池限制，并发过高会触发 rate limit 或连接超时。

主 Agent 启动 subagent 后，等待结果时设置统一超时。超时后先重试一次，再失败就降级为“返回空结果 + 错误原因”，不要让主流程卡死。

### 4. 结果只回传结构化摘要

不要让 subagent 把完整对话历史或完整工具输出回传。约定输出 JSON，只包含：

- 结论摘要；
- 关键证据（文件路径、行号、链接）；
- 置信度；
- 失败信息。

大结果写文件或对象存储，subagent 只回传路径和摘要。这样主 Agent 的上下文不会膨胀。

### 5. 合并与回退

主 Agent 收集所有 subagent 结果后，做两件事：

- 合并：按任务 ID 对齐结果，保留冲突项，交由主 Agent 做最终决策；
- 回退：某个 subagent 失败时，用缓存结果或忽略该维度，不影响其他并行任务。

## 踩坑点

- **上下文复制**：如果每个 subagent 都复制主 Agent 的完整上下文，token 成本会翻倍。只传任务输入和必要背景。
- **MCP 连接数**：多个 subagent 同时调用同一个 MCP server，容易触发限流。按工具类型设置并发上限。
- **结果丢失**：subagent 返回大段自然语言时，主 Agent 汇总容易漏信息。强制 JSON 输出。
- **嵌套爆炸**：subagent 再调用 subagent，层级一多就不可追踪。限制最大深度 1～2 层。
- **权限过大**：继承主 Agent 全部工具是危险做法。按任务白名单授权。

## 可复用建议

- subagent 保持无状态、单职责，每次调用独立；
- 输入/输出用 schema 校验，失败时能快速定位；
- 并发从 2 开始，稳定后再逐步提高；
- 大结果写文件/对象存储，只回传路径；
- 每个 subagent 调用都带 trace_id，日志可追踪；
- 先串行跑通，再改并行，避免“并行放大错误”。

## 总结

subagent 编排的核心不是“同时跑多个 AI”，而是用工程手段控制并行边界。任务拆得清、结果回得小、失败隔得开，并行才能稳定产生收益。在 OpenClaw/Agent/MCP 这类工具链里，先把单个 subagent 调好，再考虑并行，通常比一上来就画复杂编排图更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/04ac6a66521d5d36.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/98cdabc319909e04.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/58cc7754aea3e563.png)

