---
title: OpenClaw 多模型路由实践：如何让 GPT 干重活，本地模型打辅助
feedId: 31821
source: 综合讨论
publishedAt: 2026-08-06
---

# OpenClaw 多模型路由实践：如何让 GPT 干重活，本地模型打辅助

## 背景：为什么需要多模型路由

在基于 OpenClaw 构建 Agent 或自动化管线时，模型的选择不再是“一把梭”：我们既希望在复杂推理、多模态理解上借用 GPT-4o 的强能力，又需要面对大批量、低延迟、数据敏感的日常任务时，将成本压到几乎为零。一个典型的场景是：用本地 Ollama 部署的 Llama 3.1 或 Qwen 处理 90% 的文本结构化任务，只在关键时刻把难题扔给远程大模型。

OpenClaw 作为工具链，本身支持多模型后端接入——可同时配置 OpenAI-compatible API、Ollama、甚至通过 MCP 桥接的私有服务。但模型选得对不对，直接影响成本、稳定性与输出质量。几个月实践下来，我们摸索出了一套在 OpenClaw 中落地模型路由的工程方案，这里记录核心思路、配置方式及真实踩坑。

## 问题拆解：什么时候用 GPT，什么时候用本地模型

决策的关键不在于“模型强不强”，而在于任务到底需要什么。我们通常从四个维度评估：

1. **任务复杂度**：需要多步推理、代码生成、复杂 JSON 填充 → 倾向远程强模型。
2. **上下文长度**：超过 8K token 的文档处理，本地 7B/13B 模型往往注意力散掉，需 GPT 长窗。
3. **数据敏感性**：涉及用户 PII、内网文档 → 强制本地，避免数据出域。
4. **延迟与成本**：要求 <500ms 响应，或吞吐量 >10 QPS → 本地优先；偶尔调用、对延迟不敏感 → GPT。

更具体地，我们画了一条经验线：

- **纯提取/总结/翻译**（字数 <2k）：本地模型足够，准确率可达 GPT-4o 的 85%+。
- **多条件筛选、意图分类、短文本标签**：本地模型表现稳定，甚至可微调专用模型。
- **复杂指令跟随、工具调用 (function calling)、多轮对话管理**：目前本地模型与 GPT-4o 差距明显，失败率高，建议走远程。
- **图像理解、音视频分析**：除非本地部署了多模态模型且经过充分测试，否则一律用 GPT-4o / Claude。

## 在 OpenClaw 中落地的具体做法

OpenClaw 的 `ModelRouter` 允许通过 YAML 配置文件或 Python decorator 定义路由规则。我们采用了“规则优先 + 置信度分级”的策略，而不是让一个“裁判模型”去实时判断（那样增加一次调用成本）。

### 1. 后端配置

```yaml
# model_backends.yaml
backends:
  gpt4o:
    type: openai
    model: gpt-4o-2024-08-06
    api_key_env: OPENAI_API_KEY
    max_retries: 2
    timeout: 30s
  local_qwen:
    type: ollama
    model: qwen2.5:14b
    base_url: http://localhost:11434
    timeout: 10s
  local_llama:
    type: ollama
    model: llama3.1:8b
    base_url: http://localhost:11434
    timeout: 10s
```

### 2. 路由规则编写

我们在路由层定义了明确的开关：根据标签（task_type）、内容特征、上下文长度等分流。

```python
from openclaw.router import rule, RoutingContext

@rule
def sensitive_data_guard(ctx: RoutingContext) -> str:
    # 包含身份证号、手机号等模式，强制本地
    if ctx.metadata.get("contains_pii"):
        return "local_qwen"
    return None

@rule
def task_based_routing(ctx: RoutingContext) -> str:
    task = ctx.metadata.get("task_type")
    if task in ("summarize", "translate", "entity_extract"):
        # 短文本走本地
        if ctx.prompt_length < 2000:
            return "local_llama"
    if task in ("code_generation", "multi_step_reasoning", "tool_use"):
        return "gpt4o"
    return None

@rule
def fallback_long_context(ctx: RoutingContext) -> str:
    if ctx.prompt_length > 8000:
        return "gpt4o"
    return None
```

`RoutingContext` 会携带 prompt、长度、用户自定义的 metadata（如 task_type、contains_pii）。所有规则组成链，第一条非空返回值即为选定后端。如果没有命中任何规则，Router 会回退到全局默认模型（我们设为本地 Llama）。

### 3. 置信度分级（可选）

对于分类、意图识别等任务，我们还利用本地模型输出 token 概率。若生成结果的置信度低于阈值（比如 <0.75），则自动升级到 GPT 重新执行。这样既节省成本，又能兜底准确率。

### 4. 工具调用特化

当 Agent 需要通过 MCP 调用外部工具时，我们几乎统一走 GPT-4o。早期尝试用本地模型 function calling，频繁出现参数类型错误、缺字段问题；即使 fine-tune 后的模型，在复杂工具选择上仍不稳定。因此，凡是 OpenClaw 工具链路，默认路由到 GPT，只有一些极简单、无参数的工具（如获取当前时间）才放开给本地模型。

## 踩坑记录

**坑1：本地模型对 system prompt 格式极度敏感**  
例如 Qwen 对话模板要求显式 `<|im_start|>system` 标记，如果 OpenClaw 拼接时使用了 OpenAI 的 messages 格式，本地模型可能产生大量乱码或拒绝回答。解决：在后端适配层做模板映射，或使用 Ollama 的原生 chat API 而非 completion。

**坑2：路由规则过于简单，反而推高成本**  
一开始我们用“只要涉及 JSON 输出就用 GPT”，结果大量简单结构化提取任务（如“把这段话变成 name:… age:… 格式”）本可以本地完成，却浪费了预算。后来细化规则，抽取出“轻量格式化”子类型，路由到本地。

**坑3：并发下的资源竞争**  
本地模型跑在单张消费级 GPU 上，当多个 Agent 同时触发本地路由时，推理排队导致延迟暴涨，进而触发上游超时。我们不得不引入本地队列限流，并对延迟敏感路径直接路由到 GPT（成本换确定性）。

**坑4：成本估计不透明**  
最初只按 prompt 长度估算 token，忽略了输出 token 和工具调用的额外消耗。建议在 OpenClaw 中接入 cost tracker，记录每次调用的实际 token 量，便于离线分析优化规则。

## 可复用的工程建议

1. **从任务标签切入，而非模型比分切入**：在业务层给每个请求打上任务标签（如 `summarize`, `code`, `tool_use`），让路由规则可读可维护。
2. **默认本地，白名单远程**：将本地模型设为全局默认，只对明确需要强能力的任务放行 GPT。这样新接入的自动化流程不会意外产生高额费用。
3. **监控“升级率”**：统计本地路由命中后因置信度低或格式错误而升级到 GPT 的比例。如果某类任务升级率 >20%，考虑直接修改规则，将其划为远程。
4. **本地模型选型别贪大**：14B/7B 在很多轻量任务上已经够用，且响应时间可控。没必要上 70B 而导致并发吞吐下降。
5. **把工具调用单独拎出来**：目前阶段，构建在 MCP 上的工具链尽量走远程；同时保持对本地工具调用能力的持续评估，能力成熟后再分流。

## 总结

多模型路由在 OpenClaw 中不是一个炫技的“智能路由”，而是实实在在的工程手段：用最小成本满足绝大多数请求，把省下的预算精准投放在复杂任务上。经过几个星期的规则迭代与监控，我们将 GPT 调用量压缩到总请求的 8%，但保持了整体任务成功率不降反升。对于每天都在用 Agent 做自动化处理的团队来说，这种改动带来的成本优化是立竿见影的。

建议所有正在用 OpenClaw 的团队，尽快把路由规则显式化，别再手动判断“这次该用哪个模型”——那正是工具该代劳的事。

---

