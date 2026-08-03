---
title: OpenClaw 多模型路由：什么时候该用 GPT，什么时候该用本地模型
feedId: 31525
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

OpenClaw 的 `config` 里可以同时挂多个 provider：OpenAI、Ollama、vLLM 等。大多数人的用法却只有两种极端：全程走 GPT，或者一改配置全切本地模型。前者月底看账单肉疼，后者跑两天发现任务成功率掉了一截。多模型路由不是"二选一"，而是按任务特征把请求分派到成本曲线最优的位置。

## 问题

Agent 的工作流通常包含规划、工具调用、结果抽取、总结这几个环节。它们对模型的要求完全不同：长链路工具编排需要强推理能力，而"把这段文本转成 JSON"这种活，7B 模型完全够用。用 GPT-4o 处理后者是浪费，用本地模型处理前者大概率翻车。

## 做法：按负载类型分派

先把任务分成三类：

- **A 类（规划/推理）**：任务分解、多步工具编排、长上下文总结 → GPT-4o
- **B 类（执行/抽取）**：单轮工具调用、结构化输出、字段提取 → GPT-4o-mini 或本地模型
- **C 类（批量/离线/隐私）**：批量改写、检索重排、含敏感字段的数据 → 本地模型（qwen2.5:7b、llama3.1:8b，通过 Ollama/vLLM 暴露 OpenAI 兼容端点）

路由判据按顺序执行：

```yaml
# 伪配置示意
models:
  planner: { provider: openai, model: gpt-4o }
  lite:    { provider: openai, model: gpt-4o-mini }
  local:   { provider: ollama, model: qwen2.5:7b, base_url: "http://localhost:11434/v1" }
```

```
1. 任务带 privacy 标记（如 pii/finance）→ 强制 local
2. 工具调用数 > 3 或上下文 > 8k → planner
3. 否则按延迟预算：能接受 2s 内响应 → lite；能接受 5s+ → local
```

实现上不需要改 OpenClaw 内核，写一个 MCP tool 包装 `route(task)` 函数，在 agent loop 的 dispatch 层调用即可。更简单的方案是**按角色路由**：planner 固定走远程强模型，executor 固定走本地模型，中间结果过一层 schema 校验。

## 踩坑点

1. **本地模型的 function calling 格式不统一**。Qwen 系部分版本需要特定的 chat template 才能正确解析 tool calls，否则 OpenClaw 拿到的是空参数。上线前必须跑一个包含 3 次工具调用的 smoke test。
2. **上下文窗口是硬约束**。8k 上下文的本地模型在长日志累积后会出现重复 token，别让它处理完整会话历史。折中方案：让 GPT-4o-mini 做 inter-step summary，把压缩后的 context 再喂给本地模型。
3. **"便宜"可能是假象**。CPU 推理的本地模型单请求 5-10 秒，如果它处于关键路径上，整体延迟反而比 API 高。本地模型只放非关键路径或并行分支。
4. **质量污染是双向的**。本地模型的 noisy JSON 被 planner 当输入后，可能导致下一轮规划整个崩掉。在路由层加 JSON schema validation，不合格就重试或降级到 lite。

## 可复用建议

- 建一个 30 条任务的基线评估集，记录 pass/fail、耗时、token 成本。每一次调整路由策略前先跑基线，别凭感觉切模型。
- 默认路由表直接抄：复杂规划 → gpt-4o；中间层抽取 → gpt-4o-mini；批量/改写/检索后处理 → local。这个组合在成本和成功率上比较均衡。
- 给本地模型加 fallback：超时或报错时自动降级到 API；反过来 API 429 时排队到本地。双向兜底能让链路更稳。
- 把本地模型当"旁路校验器"：不替代 GPT，而是用它 cross-check GPT 的工具调用结果。对于非关键字段，这种冗余校验比直接换模型更稳。

## 总结

多模型路由核心不是"本地替代云"，而是让每类模型只做它擅长的事。先测量，再路由。评估集、路由表、fallback 链路，这三样比选哪个模型重要得多。

---

