---
title: OpenClaw 多模型路由实践：何时用 GPT 何时用本地模型
feedId: 31264
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：一只脚踩在 GPU 贫瘠地上，另一只脚还想省钱

在 OpenClaw 构建的 Agent 工作流里，模型调用是最大的持续开销。跑在云上的 GPT-4o/Claude 系列推理强、工具调用（Function Calling）格式稳定，但当你的 Agent 每天要处理数千次简单意图识别、文本润色或 MCP 数据读取时，账单会让团队主动开始翻本地服务器的配置单。

本地模型（通过 Ollama 或 vLLM 部署）成本几乎固定，延迟可控，隐私数据不出域。但直接用 Qwen2.5-7B 顶替所有 GPT 调用，很快会遇到工具调用 JSON 错乱、复杂指令跟随失败、上下文过长时遗忘初始目标等典型问题。

于是自然出现一个工程需求：在 OpenClaw 的任务管道里，**让不同类型的 prompt 走到最合适的模型上**。本文记录我们在 OpenClaw + MCP 插件体系中落地多模型路由的做法、踩坑点及可复用的配置思路。

---

## 问题：所有 prompt 一视同仁的代价

未做路由时，Agent 的每一个模型调用都发给同一个后端（通常就是 GPT-4o-mini 或 local-model）。主要矛盾包括：

- **高延迟任务不需要大模型**：MCP 工具返回的 JSON 摘要，只需做一次“是否包含异常字段”的判别，本地 7B 模型 200ms 即可完成，GPT 却需要 1.2s 且额外产生令牌开销。  
- **复杂推理本地模型掉链子**：多步推理、需要组合多个工具结果并进行逻辑判断时，7B~14B 模型经常在 ReAct 循环中产生幻觉或跳过关键验证步骤。  
- **工具调用格式参差不齐**：本地模型对 OpenAI 工具调用格式的稳定度远不如 GPT 系列，会导致 Agent 流程中断，需要额外重试与解析补丁。

因此，路由的目标不是“用本地模型替代 GPT”，而是将确定性的、容忍偶尔失败的任务下沉，把高价值推理留给云端模型。

---

## 做法：在 OpenClaw 中插入一个轻量路由决策节点

OpenClaw 的 pipeline 允许自定义 `step` 节点，我们通过一个 `model_router` 步骤实现分发。核心思路是根据任务元数据（task intent）决定后端，而不是在 prompt 内容里做复杂判断。

### 1. 定义任务类型路由表
在 `agent.yaml` 中增加以下配置片段：

```yaml
steps:
  - name: model_router
    type: switch
    condition: $task.intent
    cases:
      intent_classification:
        model: local_qwen_7b
      mcp_read_summary:
        model: local_qwen_7b
      tool_result_validation:
        model: local_qwen_14b
      multi_step_reasoning:
        model: gpt-4o-mini
      complex_code_generation:
        model: gpt-4o
      fallback:
        model: gpt-4o-mini
```

### 2. 如何获得 `task.intent`
在 Agent 入口处增加一个极轻量的意图分类步骤，也走本地模型。给本地模型的 prompt 严格控制输出为枚举值：

```
Classify the user request into exactly one of: intent_classification, mcp_read_summary, tool_result_validation, multi_step_reasoning, complex_code_generation. Return only the label.
```

这一步使用量化版 Qwen2.5-1.5B 或 3B，延迟低于 100ms，令牌消耗极低，分类准确率可达 90% 以上。误分类的少数情况由 `fallback` 兜底，将请求升级到 GPT-4o-mini，整体效果可接受。

### 3. 为本地模型适配工具调用
对于需要本地模型输出工具调用指令的场景（如 `tool_result_validation`），我们不直接依赖原生 function calling 能力，而是采用 constrained JSON 生成。在 Ollama 端通过 `format: json` 并配合 Pydantic 解析，避免本地模型产生非法 JSON。OpenClaw 的插件机制允许我们拦截模型输出并过一遍 schema 校验，失败时自动重试一次或回退到 GPT。

### 4. 可观测性与成本监控
在路由节点前后埋入 `log_model_choice` 步骤，记录每次选择的模型、延迟、令牌数。配合 Grafana 面板，一周内就能看到：哪些意图分类被误判、本地模型工具调用成功率、每日 API 费用分布。基于数据再调整路由规则，比凭感觉改配置高效得多。

---

## 踩坑记录

- **本地模型上下文窗口远比 GPT 小**：Qwen2.5-7B 原版上下文 32k，Agent 拼接的历史对话 + 工具结果经常会超限。必须对进入本地模型的输入做截断，只保留最近 3 轮工具调用结果，而不是照搬发给 GPT 的完整上下文。  
- **量化模型会产生“迷之自信”的幻觉**：在验证工具返回数据合法性时，4-bit 量化版可能会把明显异常值判断为正常。建议 key-only 的验证任务使用非量化或 8-bit 模型，或者在 prompt 中强制要求输出判断依据，由外部规则再次校验。  
- **工具调用格式补丁不是银弹**：即便用 constrained JSON，本地模型有时仍会输出多余文字包裹 JSON，需要在解析器里做容错处理（如用 regex 提取首个 `{` 到最后一个 `}` 的片段）。这会轻微增加延迟。  
- **盲目增加路由规则导致维护灾难**：一开始我们为每一种 MCP 工具设了一条路由规则，两周后 `agent.yaml` 膨胀到 500 行。后来收敛为按意图而非工具名路由，当新插件接入时，只需归入已有意图类别。

---

## 可复用的建议

1. **从两个模型开始，而不是四个**：先只设定本地模型（7B/14B）+ GPT-4o-mini 两路，把意图分类和简单摘要下沉，其余仍然走 GPT。等稳定两周后再考虑第三路。
2. **路由条件尽量基于结构化元数据**：如 `task.intent`、`task.required_tools`，避免在路由节点里再做 NLP 判断，否则路由本身的成本会吃掉节省的好处。
3. **本地模型选型建议**：优先使用官方指令微调版本（如 Qwen2.5-14B-Instruct），其在工具调用格式遵循上明显优于 base 模型。部署时使用 vLLM 可提高并发吞吐，少量并发可用 Ollama 更简单。
4. **建立自动降级机制**：当本地模型连续失败 2 次或输出无法解析时，自动将当前任务升级到 GPT 模型，并上报异常事件以便后续分析。
5. **用 MCP 描述规范对齐**：确保所有 MCP 工具描述参数明确、举例充分，本地模型对格式的遵循度高度依赖工具描述质量。

---

## 总结

多模型路由不是“因为穷才用本地模型”的权宜之计，而是让不同能力等级的模型各司其职的工程常态。在 OpenClaw 体系里，通过一个简单的意图分类 + switch 路由就能将 60%-70% 的简单调用从云端抽离出来，而复杂推理依然享有大模型的推理能力。关键在于保持路由逻辑的简洁、可观测以及自动降级，避免陷入规则膨胀的陷阱。

如果你正在 OpenClaw 上跑着 24 小时不间断的 Agent 任务，不妨先用一个本地 1.5B 分类器试水，观察一周 cost breakdown，很可能就会找到第一批值得下沉的任务。

---

