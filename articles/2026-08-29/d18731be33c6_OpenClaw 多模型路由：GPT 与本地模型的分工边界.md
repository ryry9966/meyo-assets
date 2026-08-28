---
title: OpenClaw 多模型路由：GPT 与本地模型的分工边界
feedId: 35118
source: 综合讨论
publishedAt: 2026-08-29
---

# OpenClaw 多模型路由：GPT 与本地模型的分工边界

## 背景

在 OpenClaw 上接 MCP、插件和自动化任务后，很容易进入一个状态：所有请求都走 GPT 类云端模型。好处是输出稳定、工具调用可靠，坏处是 token 消耗、延迟和费用上涨得很直接。也有另一类实践是全量切到本地模型，比如通过 Ollama 或 vLLM 暴露 OpenAI 兼容接口。短任务表现不错，但一旦遇到复杂规划、多步工具调用或长上下文，本地模型容易返回不合规 JSON、漏掉工具参数或直接跑偏。

所以多模型路由不是“选一个”，而是把任务分层，让云端模型做高价值、高可靠性要求的部分，本地模型做低复杂度、高隐私或可重试的部分。

## 问题

什么时候该用 GPT，什么时候该用本地模型？工程上可以看四个维度：

1. **任务复杂度**：单轮分类、摘要、格式整理、意图识别，通常适合本地模型；多步推理、跨系统规划、复杂工具编排，适合云端模型。
2. **工具调用**：涉及 MCP tool call、严格 JSON 参数生成时，优先云端模型，除非本地模型已经验证过 function calling 稳定性。
3. **上下文长度**：超过本地模型可靠窗口的任务，尤其需要密集引用长文档的任务，走云端。
4. **数据敏感与成本**：内部数据、日志、不需要联网的任务，本地模型更合适；面向最终用户且不能频繁重试的任务，云端更稳。

## 做法/步骤

在 OpenClaw 里可以维护两个或多个模型 profile，例如 `cloud_profile` 指向 GPT-4o-mini 或 GPT-4.1，`local_profile` 指向本地 Qwen、Llama 或其它 7B-14B 模型。然后按规则路由。下面是一个思路示例，字段名以实际版本为准：

```yaml
routing:
  - name: tool_heavy
    match:
      has_tool_calls: true
    profile: cloud_profile
  - name: long_context
    match:
      estimated_tokens: ">8000"
    profile: cloud_profile
  - name: strict_json
    match:
      output_format: json_schema
      risk: high
    profile: cloud_profile
  - name: local_fast
    match:
      has_tool_calls: false
      estimated_tokens: "<=512"
    profile: local_profile
  - name: internal_data
    match:
      data_scope: internal
      has_tool_calls: false
    profile: local_profile
```

如果当前 OpenClaw 版本没有这么细的内置路由字段，可以把路由做成一个前置调度节点：先用本地小模型对输入做一次快速分类，输出 `route: local` 或 `route: cloud`，再交给对应 profile。这种方法多一次本地推理，但规则清晰、可观测性好。调度节点本身应使用低延迟、低成本的模型。

## 踩坑点

第一，本地模型的工具调用没有想象中稳。7B/8B 模型偶尔会生成多一个字段、漏掉必填参数，或把 JSON 包在 markdown code fence 里。工具链路上来就失败，重试成本很高。建议工具调用默认走云端，本地模型只做无工具调用或低风险工具调用。

第二，输出格式漂移。即使不用工具，让本地模型输出 JSON 或固定结构时也可能出现多余解释、单引号、编码问题。可以设置 `temperature=0`，用固定 prompt 模板，配合推理后端支持的 JSON schema 约束；但不要假设约束一定生效，要在端上做校验。

第三，上下文长度不能只看模型标称值。并发请求下，本地 GPU 显存占用会被 KV cache 放大。一个 8k 上下文的模型，在并发时可能无法稳定吃满 8k 输入。建议给本地模型预留更多余量，比如把本地路由阈值设在 4k-6k。

第四，路由规则顺序敏感。`tool_heavy`、`long_context`、`strict_json` 可能同时命中，谁先谁后会影响最终 profile。应先放高优先级约束，例如安全、输出格式、工具调用，再放成本优化规则。

## 可复用建议

1. 先做最小的基线测试：挑 20-50 条真实任务，比较本地模型和云端模型在 **准确性、延迟、格式错误率、单条成本** 上的差异。不要凭直觉把任务分给本地模型。
2. 本地模型先承担可重试、非面向最终用户的任务，例如日志分类、消息预处理、内部数据提取、摘要初稿。稳定后再扩大范围。
3. 所有路由决策打日志，记录调用了哪个模型、命中哪条规则、是否失败重试、耗时多少。路由规则应该根据日志调整，而不是一次性写好。
4. 数据敏感与复杂度要分开看。内部数据适合本地，但如果任务本身复杂，可以先做脱敏再走云端，或者只用本地模型做脱敏/分段，云端做综合推理。
5. 为本地模型准备严格输出校验。所有来自本地模型的 JSON 都先过 schema 校验，失败后降级到云端重试，并在日志中标记 `local_json_invalid`。

## 总结

OpenClaw 的多模型路由本质是任务边界管理：云端模型负责工具调用、长上下文、复杂推理和高可靠性输出；本地模型负责低复杂度、高隐私、可重试、高吞吐任务。可以先从“是否工具调用 + 上下文长度 + 输出格式要求”三个条件开始，再逐步加入数据敏感和延迟约束。路由的价值不是把所有请求本地化，而是让每次失败的代价更小、每次的 token 花得更清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c52e86dad9ba4c5f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c07628d7abf358c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/611f19b0931d4e5e.png)

