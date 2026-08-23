---
title: OpenClaw 多模型路由：GPT 负责关键决策，本地模型消化高频杂活
feedId: 34357
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

OpenClaw 的 Agent 链路很少是单次模型调用，而是“规划—执行—校验”的多次调用。一个任务里可能同时包含：

- 理解用户意图
- 调用 MCP 工具
- 抽取结构化字段
- 聚合多步结果
- 生成最终回复

如果所有环节都走 GPT，成本和首字延迟会迅速上升；如果全部压到本地模型，复杂工具调用和长上下文推理又容易失败。实际工程里更需要的是：**在哪个环节用哪个模型更稳、成本更可控**，而不是“一刀切”。

## 问题

多模型路由容易做成散落在各个插件里的 if-else。比如某个插件写死用 GPT，另一个插件写死用 Ollama，过一段时间就没人知道当前请求会走哪条路径。更合理的方式是在 OpenClaw 里加一层统一路由，按任务类型、上下文长度、隐私要求做分发。

下面把落地拆成四步：任务分级、模型注册、路由规则、降级策略。

## 做法 / 步骤

### 1. 先给任务打标签，不急着写路由

把 OpenClaw 中所有模型调用点先分成三类：

- **A 类：强推理 / 工具调用 / 规划**  
  例如多步工具选择、复杂 MCP schema 理解、长上下文聚合。这类交给 GPT-4o、Claude 等云端强模型。

- **B 类：抽取 / 分类 / 格式转换 / 短摘要**  
  例如意图分类、实体抽取、JSON 格式化、短文本摘要。这类可以下沉到本地 Qwen、Llama、DeepSeek 等模型。

- **C 类：不确定 / 长上下文 / 多步结果聚合**  
  先走 GPT，等稳定后再考虑是否下沉。

### 2. 统一封装模型调用

不要在插件里分别调 OpenAI SDK 和 Ollama client。可以在 OpenClaw 里写一个 `model_router.py`，或者在 policy 层定义规则，对外暴露统一接口：

```python
complete(task_type="intent_classification", messages=msgs)
complete(task_type="tool_planning", messages=msgs)
```

这样后续切换模型、加降级、加埋点都只改一个地方。

### 3. 路由规则示例

```yaml
router:
  default: gpt-4o-mini
  rules:
    - task_type: intent_classification
      model: local_qwen2_7b
      temperature: 0
    - task_type: entity_extraction
      model: local_llama3_8b
      fallback: gpt-4o-mini
    - task_type: tool_planning
      model: gpt-4o
      min_confidence: 0.8
    - task_type: long_context_summary
      model: gpt-4o
      force_cloud: true
```

这里的关键不是规则多少，而是每条规则都有明确的 task_type 和 fallback。

### 4. 给本地模型输出加约束

本地模型在 JSON 输出和工具调用格式上容易飘。建议：

- 使用 constrained decoding，例如 Outlines、llama.cpp grammar
- 或者生成后用 Pydantic 校验，失败自动重试一次
- 再失败就升级到云端模型

不要把本地模型的裸输出直接传给下一个 MCP 工具。

### 5. 加观测

返回结构里带上：

```json
{
  "model_used": "local_qwen2_7b",
  "latency_ms": 842,
  "fallback_reason": null
}
```

没有这些字段，后续没法判断路由是否合理，也没法做成本统计。

## 踩坑点

- **本地模型不一定省时间**  
  7B/8B 模型在 CPU 上跑，抽取 100 条短文本可能要几十秒，而 GPT-4o-mini 可能更快。下沉之前先压测本地推理速度，否则可能“省了 token，赔了延迟”。

- **工具调用稳定性是最大坑**  
  本地模型对复杂 MCP schema 的理解容易出错，尤其是多工具选择、参数名嵌套。A 类任务不要下沉，或者只允许本地模型调用一两个简单工具。

- **上下文窗口**  
  很多本地模型实际可用窗口小于标称，长 system prompt 加多轮工具结果容易截断。长上下文聚合不要走本地模型。

- **路由规则过细**  
  一开始不要定义几十条规则，按 task_type 粗分即可。规则过多，维护成本会超过模型成本。

- **隐私不等于本地**  
  本地模型虽然数据不出域，但如果模型权重来源不明、日志里记录了原始输入，依然有泄露风险。敏感数据要在调用前脱敏，日志只留 task_id 和哈希。

## 可复用建议

- **先找高频低风险环节下沉**  
  用 OpenClaw 日志统计各 task_type 的调用次数、token 消耗、延迟。优先优化调用量大、风险低的环节，例如意图分类、实体抽取。

- **用降级链代替硬编码**  
  本地模型失败或低置信时自动升级到 GPT；高价值任务先 GPT，跑稳后再本地化替换。这样随时可以回退。

- **独立限制本地模型并发**  
  Agent 进程和本地模型在同一台机器时会抢 CPU/GPU。给本地推理服务限制并发，或者放到独立 worker，避免互相拖累。

- **每次切换前准备小型 eval set**  
  20–50 条典型任务，对比准确率、格式合规率和延迟。不要凭感觉换模型。

## 总结

OpenClaw 多模型路由，本质是把模型当成不同成本的执行单元来调度。GPT 适合做规划、工具路由和长上下文决策；本地模型适合做高频、结构清晰、可校验的杂活。

关键不是省掉所有 GPT 调用，而是让每次调用都出现在合适的位置，并且可观测、可回退。这样系统不会越做越脆弱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/13c3d319e285a8a7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/de8dd1ab7f737470.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ed241cacad0e8f71.png)

