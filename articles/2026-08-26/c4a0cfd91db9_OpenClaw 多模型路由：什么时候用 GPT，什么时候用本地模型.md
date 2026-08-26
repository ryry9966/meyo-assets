---
title: OpenClaw 多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 34843
source: 综合讨论
publishedAt: 2026-08-26
---

# OpenClaw 多模型路由：什么时候用 GPT，什么时候用本地模型

## 背景

OpenClaw 的 Agent 执行链里，模型调用通常是大头成本。全量走 GPT，账单长得快；全量走本地模型，又容易在工具调用、长上下文、复杂推理上翻车。多模型路由不是为了技术炫技，而是为了在稳定性、成本和延迟之间做分区。尤其在 MCP、插件、自动化任务混跑的场景下，不同任务对模型能力的要求差异很大。

## 问题

实际项目里常见两类失败。一类是本地 7B/14B 模型被安排做复杂 JSON 工具参数生成，输出字段漂移、类型错误，导致 MCP 调用失败，然后错误只表现为“invalid json”，定位很慢。另一类是把简单意图识别、短摘要、敏感数据清洗也打给 GPT，成本高、延迟也不划算。如果路由策略只按模型名称硬编码，任务一多就失控。

## 做法/步骤

第一步，先做任务分类。我通常把任务分成：简单意图识别、短文本摘要、结构化提取、工具参数生成、长上下文总结、代码生成、隐私敏感数据清洗。不同任务对格式、推理、上下文的依赖不一样。

第二步，在 OpenClaw 的 model router 里加一个策略层，不要散落在插件或 Agent 代码里。示例如下，实际字段可按自己的 OpenClaw 版本调整：

```ts
function route(task) {
  if (task.private || task.localOnly) return 'local'
  if (task.kind === 'intent' || task.kind === 'short_summary') return 'local'
  if (task.kind === 'tool_args' && task.complexity === 'high') return 'gpt'
  if (task.estimatedTokens > 8000) return 'gpt'
  if (task.budget === 'low') return 'local'
  return 'gpt'
}
```

第三步，本地模型选型。优先选支持 function calling 且量化后显存可控的模型，例如 13B/14B 的 GGUF/AWQ 版本。如果只用 CPU 跑，需求降到 3B/7B。本地模型用独立推理服务部署，OpenClaw 通过 HTTP 调用，并限制 max_tokens、temperature 0.1–0.3，尽量开 JSON grammar 或 constrained decoding。

第四步，GPT 集中用于高复杂度工具调用、长上下文、多步规划。常用 prompt 做缓存，避免无意义重复调用。

第五步，做 fallback。本地模型解析失败或超时，自动降级到 GPT；GPT 调用失败时，降级到本地兜底，但需要给本地模型额外上下文和示例，不能直接复用原 prompt。

## 踩坑点

- 本地模型 JSON 输出不稳定，尤其嵌套对象和数组元素顺序。不要假设它会输出合法 JSON。必须用 JSON schema 约束，或解析前做 repair。OpenClaw 中 MCP 工具参数解析失败时，错误经常被吞掉或只显示解析错误，要提前接好日志。
- 本地小模型对系统提示词很敏感。直接用 GPT 的 system prompt 会让本地模型跑偏。需要给本地模型写更直接的指令，示例控制在 1–2 个。
- 并发与 GPU 显存：本地模型同时响应多个 Agent 请求会排队，延迟可能从 1 秒涨到十几秒。要限制并发，最好单独部署推理服务。
- 上下文长度不要只看字符数，要按 token 估算。简单任务如果带着一堆 MCP 工具 schema，本地模型可能直接吃不下。
- 不要频繁切换模型温度等参数。路由决策要确定性优先，随机切换会造成线上行为漂移。

## 可复用建议

- 用一个小的评估集，每次换本地模型前先跑 30–50 个真实任务，记录解析成功率、工具调用正确率、延迟。
- 在 OpenClaw 中给每次请求打上 model、task_type、latency、success、cost 标签，路由策略根据数据调，而不是凭感觉。
- 将任务分级：P0 固定 GPT；P1 本地优先，失败转 GPT；P2 只走本地。
- 对本地模型设置更严格的 timeout 和 retry，不要和 GPT 共用同一套超时。
- 把路由策略写成纯函数，方便单元测试，避免散落在插件和 Agent 代码里。

## 总结

多模型路由不是让本地模型替代 GPT，而是把确定性高、低复杂度、隐私敏感的任务从 GPT 那里分流出来。工程上先保证稳定，再降本。一个可用的判断是：如果本地模型在某个任务上需要大量 prompt hack 才能稳定，那就先别硬上，等评估通过再切。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/10b14fa61c8ea9ce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/cc0c0c76ab38b838.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/eb05ee17a62d79a1.png)

