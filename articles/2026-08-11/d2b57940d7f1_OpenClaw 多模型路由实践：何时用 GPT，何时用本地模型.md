---
title: OpenClaw 多模型路由实践：何时用 GPT，何时用本地模型
feedId: 32625
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：为什么需要多模型路由

在基于 OpenClaw 搭建 Agent 自动化管线时，一个常见的撕扯是：全用 GPT‑4 类云端大模型，效果虽好但成本与延迟都很难控；全部切到本地模型（通过 Ollama、vLLM 或 TGI 部署的 7B‑70B 模型），又会因为推理能力不足导致复杂任务反复失败。纯粹一刀切的方式，在工程上几乎不可行。

OpenClaw 提供了可编程的模型路由能力，允许在同一个 Agent 或 Pipeline 中根据请求的上下文动态选择模型。结合 MCP 协议的插件调用与工具执行，我们可以把“选模型”这件事也变成一条规则链，从而在性能、成本、隐私之间找到动态平衡。

## 问题拆解：路由决策的关键维度

从工程实践看，路由不能只是“简单任务用本地，复杂任务用 GPT”这么粗线条。需要量化的维度至少包括：

- **意图与能力匹配**：该请求是摘要、翻译、分类等相对规整的任务，还是需要多步推理、代码生成、逻辑校验的复杂任务？
- **上下文规模**：需要处理的 token 数量。1k 以内的短文本与 8k 以上的长文档，对本地模型的压力完全不同。
- **隐私与数据驻留**：是否包含 PII、内部系统日志、待脱敏数据？这些无论如何不能离开本机。
- **响应时间与并发**：本地模型是否有排队？云端 API 的 rate limit 和首 token 延迟是否可接受？
- **成本预算**：针对大批量、低价值的调用，是否有硬性成本上限？

将以上维度转化为可执行的路由规则，是本文要解决的核心问题。

## 做法/步骤：在 OpenClaw 中配置模型路由

### 1. 定义模型端点
在 `openclaw.yaml` 中注册你的模型池，每个模型打上标签：

```yaml
models:
  - id: gpt-4o
    provider: openai
    api_base: https://api.openai.com/v1
    labels: { tier: premium, remote: true }
  - id: local-llama3-70b
    provider: local
    endpoint: http://localhost:8000/v1
    labels: { tier: standard, local: true, max_tokens: 4096 }
  - id: local-mistral-7b
    provider: local
    endpoint: http://localhost:8001/v1
    labels: { tier: lightweight, local: true, max_tokens: 2048 }
```

其中 `labels` 并非 OpenClaw 内置字段，但你可以在自定义 Router 中读取这些元数据。

### 2. 实现路由逻辑
OpenClaw 允许通过插件或中间件的方式注入 Router 函数。在 `router.py` 中实现一个 `decision()` 函数，根据从 MCP 插件会话中获取的上下文返回模型 ID：

```python
def decision(request_context: dict, available_models: list) -> str:
    intent = request_context.get("intent", "")
    complexity = request_context.get("complexity_score", 0.0)
    token_estimate = request_context.get("estimated_tokens", 0)
    has_pii = request_context.get("has_pii", False)

    # 隐私数据必须留在本地
    if has_pii:
        return fallback_local_model(available_models, prefer="standard")

    # 高复杂度或代码生成 → 云端强模型
    if complexity > 0.7 or intent in {"code_generation", "debugging", "math_reasoning"}:
        return "gpt-4o"

    # 长上下文但低复杂度，用支持大窗口的本地模型
    if token_estimate > 4000 and complexity <= 0.7:
        return "local-llama3-70b"

    # 简单短文本任务
    return "local-mistral-7b"
```

路由决策的关键输入——`intent` 和 `complexity_score`——可以由前置的轻量分类器或 MCP 工具生成。例如，用一个 0.5B 的分类模型快速判定意图，或基于关键词、prompt 长度生成复杂度分数，这部分开销远小于推理本身。

### 3. 通过 MCP 工具流水线串联
典型的调用链为：

```
用户请求 → MCP “意图分类”工具 → Router 决策 → 目标模型推理 → 后处理/格式化 → 返回
```

在 OpenClaw 的 Pipeline 定义中，可以将 Router 与工具调用编排为一个有向图。即便 Router 判断失误，也可以通过 `on_failure` 配置自动 fallback 到备选模型。

## 踩坑点

- **本地模型冷启动与批处理延迟**：首次调用时模型加载可能耗时几秒到十几秒。如果路由策略频繁在小模型和大模型之间跳转，冷启动开销会吃掉省下的成本。解决方式是尽量让同类任务“粘滞”在某一模型上，或使用常驻推理服务。
- **提示模板差异**：GPT 系列对 system/user 格式有明确预期，本地模型（尤其 chat‑ml 与 llama‑3 模板）可能不同。路由后如果直接复用同一套 prompt，容易得到格式错乱或截断的回答。建议在模型适配层为每个模型准备独立的 prompt 模板，并与路由逻辑解耦。
- **流式输出中断**：当请求进行到一半，如果触发 fallback 逻辑（例如检测到本地模型生成质量过低），此时切换模型会导致流式连接断开。更稳妥的做法是只在整个请求失败后才触发 fallback，而非中途切换。
- **路由决策自身的成本与延迟**：如果每次调用都走大模型去判断“该用哪个模型”，那就有套娃嫌疑。因此路由决策模型必须极度轻量，且最好共享同一个本地 instance，严禁走云端 API。
- **长上下文本地模型效果坍塌**：即使本地 70B 模型标称 8k 窗口，在接近上限时质量仍会明显下降。路由规则中需要对 token 数量留出安全边际（例如设置 70% 上限），超出部分提前截断或改用 GPT。

## 可复用建议

1. **路由决策用极轻量模型**：0.5B–1.5B 参数的分类或嵌入模型足够完成意图判别和复杂度打分，延迟低至百毫秒级，可与推理服务同机部署。
2. **为每个路由规则定义退出条件**：明确“如果不能确定，则默认走 GPT”还是“默认走本地”。通常保守策略是“不确定就用强的”，但成本会上升；可根据场景设置成本阈值后自动切换默认值。
3. **集中记录和审计所有路由决策**：在日志中保留每次路由的输入参数、最终选择的模型和耗时、是否触发 fallback。这份数据是后续调优规则和评估模型能力的唯一依据。
4. **利用缓存降低重复请求成本**：对频繁出现的确定性查询（如代码补全的常见片段、系统 help 指令），可以直接在 Router 层返回缓存结果，完全跳过大模型推理。
5. **分阶段上线**：先从 10% 流量开始使用本地模型处理明确简单的意图，观测失败率和用户反馈，再逐步扩大范围。切忌一上来就把核心流程压到本地模型上。

## 总结

OpenClaw 的多模型路由让 Agent 摆脱了“只用一个大模型”的僵化模式。通过把意图分类、复杂度打分等轻量决策前置，我们可以用 7B 甚至更小的模型处理海量低风险请求，同时把复杂推理、长上下文生成这种“刀刃任务”精准地交给 GPT‑4 级别的模型。真正的难点不在于实现路由器的几行代码，而在于长期维护成本与质量的平衡：需要不断回看路由日志、评估模型能力变化，并根据业务需要调整规则。如果你已经在用 MCP 工具链，不妨把“模型路由”当作一个新的标准化工具接入，让它成为你的 Agent 自动运维闭环的一部分。

---

