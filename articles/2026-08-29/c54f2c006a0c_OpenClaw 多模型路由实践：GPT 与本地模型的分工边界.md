---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界
feedId: 35220
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 框架里，模型不再是单一的 GPT 调用点，而是多个模型并行存在：远程闭源 API、本地 vLLM/Ollama 服务、通过 MCP 暴露的模型能力等。多模型路由的目标，是在成本、延迟、效果和隐私之间做显式取舍。

但实际项目中，很多人把路由做成了“能省则省”，先本地后云端。结果本地模型在工具调用或长上下文任务上频繁失败，反而增加重试与升级成本，最后又退化成所有请求都打到同一个模型。

## 问题

我遇到过三类典型问题：

1. 本地模型在简单任务上可用，但一到多步工具调用或 JSON Schema 严格输出时，开始幻觉参数、漏字段。
2. 远程模型全量承载，响应慢、token 成本高，涉及内部文档或用户数据时还有合规压力。
3. 路由规则写得太细，后期没人维护，最后变成一个谁都不愿动的配置黑洞。

这些问题本质上不是模型能力问题，而是缺少一个稳定的路由决策层。

## 做法/步骤

### 1. 先建立能力分层，不要只看 benchmark

把 Agent 日常任务粗分成四类：

- **判别类**：意图分类、敏感信息识别、垃圾过滤、实体抽取。适合本地小模型（7B-14B），输出空间小，生成能力要求低。
- **短生成类**：标题润色、摘要、短回复、格式修正。可用本地中模型（14B-32B），但要限制输出长度和格式。
- **复杂规划/工具调用类**：多步 Agent 规划、Function Calling、代码生成、错误恢复。默认走 GPT 或同级别远程模型。
- **长文档/高上下文类**：长会议记录分析、合同审阅、跨文档检索。远程模型优先，除非本地模型上下文窗口足够且经过验证。

### 2. 在 OpenClaw 路由层抽象策略

不要把模型名写死在每个 Agent 节点，而是用一个 router 统一决策。可以用类似下面的伪配置思路：

```yaml
router:
  default: remote
  rules:
    - task_type: [classification, entity_extraction]
      model: local-small
      upgrade_on: [parse_error, schema_invalid]
    - task_type: [summarization, rewrite]
      model: local-mid
      upgrade_on: [parse_error, schema_invalid, low_confidence]
    - task_type: [planning, tool_call, code]
      model: remote
      downgrade_on: [timeout, budget_exceeded, refusal]
    - input_tokens_gt: 6000
      model: remote
    - contains_sensitive_data: true
      model: local-mid
      disable_remote_logging: true
```

核心是**显式声明升级条件和降级条件**，而不是无条件 local first。

### 3. 配置 fallback 链

本地优先策略必须带“升级条件”。例如：

`local -> 输出校验通过 -> 返回`  
`local -> 校验失败 -> remote -> 返回`  
`remote -> 超时或明确拒绝 -> local（仅限短任务）`

在 OpenClaw 里可以用规则引擎或状态机实现。每个节点至少记录 `model_used`、`attempt`、`reason`，方便后续分析。

### 4. 监控路由质量和成本

至少记录这几个指标：

- 每个 `task_type` 的本地使用率、升级率。
- 本地模型解析失败率。
- 远程模型的平均首 token 延迟和 token 成本。
- 最终任务成功率。

建议每周看一次“升级率”。如果某类任务本地模型升级率超过 30%，说明能力边界判断错误，应该调整路由或换更强的本地模型。

## 踩坑点

1. **结构化输出不稳定**：本地模型对 JSON Schema 的遵循度参差不齐。不要直接相信“支持 function calling”的声明，要在真实数据上验证输出能否通过 schema 校验。校验失败先重试一次，再升级，避免无谓消耗。
2. **上下文长度估算不准**：用字符数代替 token 数会导致本地模型被截断。路由前用模型对应的 tokenizer 计算 token，并预留输出空间。
3. **工具调用幻觉**：很多本地模型在复杂工具调用上容易产生不存在的参数名或忽略必填字段。工具调用默认走远程，除非本地模型经过专门微调并有回归测试。
4. **并发与显存**：本地推理服务在并发下容易排队，一个 Agent 的长生成会阻塞其他请求。给本地路由设置并发上限或独立队列，必要时用更小的模型做判别任务，大模型只跑批处理。
5. **规则维护成本**：不要一开始写几十条规则。先按 3-5 类任务粗分，运行两周后根据升级率和解析失败率再细化。

## 可复用建议

- 默认把“失败升级”当作安全网，而不是追求“零远程调用”。
- 先让本地模型做判别，再做生成。判别类任务最容易拿到稳定收益。
- 给同一任务固定输出 schema，便于比较本地和远程模型的效果。
- 对敏感数据，本地模型是刚需，但要注意日志脱敏和模型推理时的数据落盘问题。
- 如果本地模型升级率持续高于 30%，说明该任务不适合本地，不要硬压成本。

## 总结

多模型路由不是简单地在 GPT 和本地模型之间切换，而是把任务特征、模型能力、成本预算和隐私边界变成一个可观测的决策系统。在 OpenClaw 里，先把能力边界划清楚，再小步切量，用升级率和解析失败率验证路由策略。稳定运行后，你会发现真正省下来的不是模型费用，而是无谓的重试、错误恢复和人工兜底时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e63138bb12b735fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fb8394ed6d217e95.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/ed13a646da8205ab.png)

