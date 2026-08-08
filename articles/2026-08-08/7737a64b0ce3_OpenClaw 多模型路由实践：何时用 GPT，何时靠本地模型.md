---
title: OpenClaw 多模型路由实践：何时用 GPT，何时靠本地模型
feedId: 32086
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在基于 OpenClaw 搭建设 Agent 或自动化流水线时，一个反复出现的问题是：**全上 GPT‑4o 账单吃不消，全用本地 7B/8B 模型又经常掉链子**。随着 Llama3、Qwen2 等本地模型能力上升，以及 Ollama/vLLM 部署的成熟，“混合路由”变成了切实可落地的优化点。

简单来说，就是让 OpenClaw 根据任务特征自动决定：  
- 简单、高频、低上下文的任务 → 调本地模型（零成本、低延迟）  
- 复杂推理、长指令、多步调用 → 调 GPT‑4o 或 GPT‑4o‑mini（更可靠）

本文记录这个过程里的做法、踩坑与可复用模板。

---

## 问题：一刀切的代价

如果不做区分，典型的痛点是：

- **成本**：大量情感分析、实体抽取、简单分类跑在 GPT‑4 上，token 消耗远高于实际需要。
- **延迟**：本地模型在低并发下可以 <200ms 返回，而 API 往返经常 >1s。
- **稳定性**：复杂任务丢给 8B 模型，一旦出现幻觉或格式错误，后续节点全部失败，排查成本高。
- **上下文窗口**：本地模型通常只有 4K~8K 上下文，长文档摘要、多轮对话很容易截断遗忘。

因此需要一个具备 **条件分支** 的调度层，这正是 OpenClaw 的强项。

---

## 做法与步骤

### 1. 接入两种模型

在 OpenClaw 的 `models` 配置里同时声明云端模型和本地模型，例如：

```yaml
models:
  - id: gpt4o
    provider: openai
    model: gpt-4o
    api_key: ${OPENAI_API_KEY}
  - id: local-llama3
    provider: ollama
    model: llama3:8b
    base_url: http://localhost:11434
```

如果本地模型使用 MCP 封装的服务（比如自托管的推理 endpoint），也可以用 `openai-compatible` provider 接入。

### 2. 设计路由节点（Classifier + Dispatcher）

不建议直接对原始输入做规则路由，因为很难靠正则区分复杂度。更稳健的做法是引入一个“分类器”节点，通常用本地模型本身来做，因为分类任务足够简单：

```
User Input → [classifier (local-llama3)] → {complexity: 1-5}
                         |
         ┌───────────────┴───────────────┐
         ▼                               ▼
[complex_tasks (gpt4o)]        [simple_tasks (local-llama3)]
```

YAML 配置片段（简化概念）：

```yaml
agents:
  classifier:
    model: local-llama3
    prompt: |
      Rate the complexity of the following user request on a scale 1-5,
      where 1 is a simple classification/extraction, and 5 is multi-step
      reasoning or long-document analysis. Respond with only the number.
    output: complexity_score

  dispatcher:
    type: condition
    conditions:
      - if: complexity_score >= 3
        then: complex_agent
      - else: simple_agent

  simple_agent:
    model: local-llama3
    prompt: "Handle this simple task: {input}"

  complex_agent:
    model: gpt4o
    prompt: "Handle this complex task carefully: {input}"
```

OpenClaw 的 DAG 引擎会根据 `complexity_score` 动态选择下一节点。

### 3. 加一层 Fallback

本地模型偶尔会输出格式错误或空响应，所以需要在 `simple_agent` 后面挂一个错误处理分支：

- 如果输出为空 / 解析失败 / 置信度低 → 自动回退到 `complex_agent`（GPT）
- 同时记录这次回退事件，方便后续分析

这可以利用 OpenClaw 的 `on_error` 或 `fallback` 机制，把本地模型当作“尽力而为”的加速器。

---

## 踩坑点

1. **分类器本身的偏差**  
   用同一个本地模型做分类时，它对自身能力边界的认知并不准，偶尔会把明显超纲的任务打低分。  
   解决：设置保守阈值（比如 ≥3 就上 GPT），并在非生产环境观察分类分布。

2. **提示格式不兼容**  
   GPT 的 system/user 消息格式与 Llama 系列不同，直接复用同一个 prompt 模板容易导致本地模型遵循度差。  
   解决：为本地模型单独维护一套精简 prompt，去掉 `###` 以外的复杂标记，尽量用 ChatML 标准。

3. **上下文截断**  
   本地模型的上下文窗口较小，如果上游节点把完整文档原文传过来，很容易超出。  
   解决：在进入路由之前，对长文本做一次摘要或截断，或只在分类节点传入首段 512 tokens。

4. **并发瓶颈**  
   当多条流水线同时打向本地模型，单卡 8B 可能成为吞吐瓶颈，延迟反而高于 API。  
   解决：为本地模型设置合理的并发限制，或使用 vLLM 多卡部署提升吞吐。

5. **监控缺失**  
   路由之后很容易忘记监控各模型的调用比例和成功/回退率。  
   解决：利用 OpenClaw 的 trace/日志 hook 统计 `model_id`、回退次数、延迟分布。

---

## 可复用建议

- **抽象路由模板**：把 classifier + dispatcher + fallback 打包成一个可复用的 sub‑graph，其他流水线直接引用。
- **按业务线灰度**：先让非关键场景（如内部报表生成）走混合路由，观察一两周再切到核心业务。
- **定期回测本地模型能力**：每升级一次本地模型版本，重新评估复杂任务通过率，动态调整阈值。
- **成本追踪**：在 OpenClaw 节点中注入 token 计数和预估成本，输出到日志，方便月末复盘。
- **本地模型预热**：启动时发几个假请求让模型加载进显存，避免首次调用延迟异常。

---

## 总结

在 OpenClaw 中做多模型路由不是炫技，而是实打实的 **成本/体验优化**。核心思路是：用便宜的本地模型处理大量琐碎任务，把宝贵的 GPT 配额留给真正需要深度推理的场景，并通过 fallback 保证整体可靠性。

只要阈值设置得当、监控到位，混合路由可以把 API 成本砍掉 40%–70%，同时不会明显影响业务指标。这套模式也适用于其他 Agent 框架，但在 OpenClaw 的 DAG + 条件分支下实现起来格外直观。

---

