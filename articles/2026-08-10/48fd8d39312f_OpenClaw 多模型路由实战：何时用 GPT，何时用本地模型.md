---
title: OpenClaw 多模型路由实战：何时用 GPT，何时用本地模型
feedId: 32381
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

用 OpenClaw 搭 Agent 时间长了，几乎都会撞上同一个问题：全用 GPT-4 太贵，全用本地模型又担心能力不够。OpenClaw 的插件和执行链本身是模型无关的，但任务复杂度差异很大——从简单的意图分类、字段提取，到多步工具调用和长程推理，一刀切的模型选择要么浪费预算，要么拉低可靠性。

OpenClaw 提供了多模型路由能力，可以在 `router` 配置中为不同的 task profile 绑定不同的模型后端（OpenAI、本地 vLLM、Ollama 等）。经过几个实际项目的折腾，我总结了一套在工程环境里可复用的决策与配置方法，重点解决“什么任务交给 GPT，什么任务留给本地模型”的问题。

## 问题拆解

路由决策的关键维度不是“模型大小”，而是：
- **能力需求**：是否需要复杂推理、函数调用、长上下文或多语言理解
- **延迟容忍度**：能否接受本地模型首次推理的冷启动时间
- **数据敏感性**：是否允许请求离开本地环境
- **调用频率**：高频调用更容易放大 API 成本
- **可靠性要求**：是否需要 fallback 兜底

在 OpenClaw 里，Agent 的一个 task 通常会经过预处理、推理、后处理几个阶段。预处理（比如意图分类、实体抽取）往往逻辑简单但调用频繁，非常适合本地模型；而核心推理和动态工具调用则更需要 GPT 这类高能力模型。

## 做法与配置步骤

### 1. 对任务分类

以实际项目为例，我把 Agent 的原子能力拆成三类：

| 类型 | 例子 | 特征 |
|------|------|------|
| 轻量路由/分类 | 识别用户指令是“查询”还是“操作” | 输入短、逻辑固定 |
| 结构化提取 | 从非结构化日志中提取 IP、时间戳 | 模式明确、需一点容错 |
| 复杂推理与工具编排 | 根据异常信息决定调用哪些 MCP 工具、分析结果并给出建议 | 多步、需要理解工具链 |

前两类基本可以由 7B-13B 的本地模型覆盖，第三类交给 GPT-4 或者能力接近的模型。

### 2. 模型能力评估

不是所有本地模型都能稳定支持 function calling（或 OpenClaw 的 tool use 封装）。实测下来：
- **Qwen2.5-7B-Instruct** 在简单分类和提取任务上表现稳定，响应快，但复杂工具调用容易输出畸形 JSON。
- **DeepSeek-Coder-V2-Lite** 对结构输出的遵守度更高，适合作为结构化提取的本地模型。
- 本地模型普遍在长上下文（>8K）后注意力衰减明显，所以需要控制输入长度。

评估方法是搭建一个迷你 benchmark：从线上日志里抽 200 条真实请求，让候选模型跑，统计成功率和 token 消耗。

### 3. 配置 OpenClaw 路由

在 OpenClaw 的 `router.yaml` 中，可以为 task 定义 profile，指定模型和参数：

```yaml
profiles:
  classifier:
    backend: ollama
    model: qwen2.5:7b
    max_tokens: 256
    temperature: 0
  extractor:
    backend: vllm
    model: deepseek-coder-v2-lite
    max_tokens: 1024
  reasoning:
    backend: openai
    model: gpt-4-1106-preview
    max_tokens: 2048
    temperature: 0.3
```

然后在 Agent 定义中引用：

```yaml
tasks:
  - name: intent_classify
    profile: classifier
  - name: log_extract
    profile: extractor
  - name: incident_diagnose
    profile: reasoning
```

这样，一个请求进入 Agent 后，不同阶段会由不同模型处理。如果某个本地模型过载或异常，还可以在 profile 里设置 fallback 到 GPT（后面会说）。

### 4. 加入 fallback 与超时控制

本地模型可能因为显存不足、OOM restart 而暂时不可用。我们在 profile 中添加 `fallback` 和 `timeout`：

```yaml
classifier:
  backend: ollama
  model: qwen2.5:7b
  timeout: 5s
  fallback:
    backend: openai
    model: gpt-3.5-turbo
    max_tokens: 256
```

超时推荐设得比 P95 延迟略高即可，不要无限等待。fallback 模型选择轻量、低成本的，避免直接跳到 GPT-4 造成预算波动。

## 踩坑记录

1. **本地模型 OOM 与显存碎片**：当多个 profile 共用同一个 vLLM 实例时，不同模型的显存占用叠加容易导致 OOM。解决方法是按负载把轻量模型放单张卡，或者用 `--gpu-memory-utilization` 限制上限，并监控显存使用。
2. **function calling 格式兼容性**：本地模型对 OpenAI 风格的 function calling 支持参差不齐，有时会返回多余文本。可以在 prompt 中强制要求 “只输出 JSON，不要解释”，并在后处理加一层校验和修复。
3. **冷启动延迟**：Ollama 加载模型需要时间，如果请求间隔超过 keep-alive，下一次请求会额外等 2-10 秒。解决办法是设置较长的 keep-alive 或使用常驻的 vLLM 服务。
4. **上下文盲区**：本地模型在处理长 prompt 时容易丢失中间信息。对于需要长文档理解的任务，应当直接走 reasoning 的 GPT profile，不要试图用本地模型压缩后再送 GPT，那样反而增加信息丢失风险。

## 可复用建议

- **高频简单任务优先本地化**：一个日均百万次的分类器，哪怕本地模型比 GPT-3.5 差 2% 的准确率，成本差异也足够覆盖修复成本。
- **工具调用与多步推理留给 GPT**：需要理解工具返回结果并决策下一步的任务，本地模型的稳定性尚不足以承担 SLO。
- **保持路由规则显式化**：不要在代码里用 `if "复杂" in task_name` 这种隐式逻辑，都收敛到 profile 配置，方便审计和调整。
- **监控分模型成本与延迟**：在 OpenClaw 里为不同 profile 打上标签，输出到监控系统，建立 per-profile 的 dashboard，避免某个本地模型退化而不自知。

## 总结

OpenClaw 的多模型路由本质上是一种“能力分层”的工程手段：用低成本的本地模型处理机械、高频的逻辑，用高能力 API 模型兜底复杂决策。这套方法不需要颠覆现有 Agent 设计，只需要合理地切分 task 并配置 profile，就能在可控风险内大幅降低 API 费用、减小外部依赖。

从几个项目的实际跟踪来看，合理的路由配置可以把 GPT-4 的调用量降低 60-80%，而端到端任务成功率几乎不受影响。关键在于前期对任务的充分拆解和对本地模型能力的诚实评估，而不是盲目追求“全本地化”或“全上 GPT”。

---

