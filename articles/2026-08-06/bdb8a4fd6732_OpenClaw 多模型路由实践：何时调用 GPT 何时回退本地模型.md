---
title: OpenClaw 多模型路由实践：何时调用 GPT 何时回退本地模型
feedId: 31800
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么 Agent 不能只绑一个模型

在 OpenClaw 的自动化管线里，最小闭环往往是“触发器 → Agent → 动作”。随着管线复杂度上升，你会很快遇到一个现实问题：**所有请求都走 GPT-4o，账单爆炸；全部用本地模型，复杂推理又翻车**。同时，某些流程涉及内部数据，不能过公网 API。多模型路由不是“选一个最好的模型”，而是根据任务特征在**成本、延迟、隐私与能力**之间做动态匹配。

OpenClaw 内置的 Router 组件可以基于规则或条件自动选择模型后端，这让我们能把“模型调度”做成工程决策，而非每次写代码 hardcode。

本文面向已在 OpenClaw 上跑过至少一个 Agent/MCP 工具链的读者，给出可直接落地的路由策略和踩坑记录。

## 问题定义：什么时候用 GPT，什么时候用本地模型？

经过十几个自动化工作流的打磨，我抽象出四个判断维度：

1. **任务复杂度**  
   需要多步推理、复杂指令跟随、长上下文关联、或生成结构化 JSON 且字段嵌套深 → **用 GPT**。  
   简单意图分类、关键词抽取、文本格式化、摘要（文章短且无精确要求） → **本地模型完全胜任**。

2. **数据敏感度**  
   包含客户 PII、内网文档、密钥等 → **强制走本地模型**（甚至可以是 CPU 推理），绝不把原文送出防火墙。

3. **延迟要求**  
   交互式场景（如聊天补全）要求 < 2 秒首 token → 本地小模型（7B-13B，GPU 推理）优于 API 往返。  
   批处理、事后分析任务可以接受 GPT 的 5-15 秒。

4. **成本约束**  
   高频低价值调用（日志摘要、触发词检测）用 GPT 纯属浪费。本地边际成本几乎为零。

一个实操例子：监控告警聚合。Prometheus 发来一堆告警，Agent 需要提取关键服务名并生成一句话总结。这种结构化提取本地 7B 模型就足够，每 30 秒轮询一次，用 GPT-4o 每月烧掉几百美金。

反过来，处理客户发来的语义模糊的工单，需要判断“是否有升级到高级工程师的意向”，涉及长文本的隐含意图推理和业务规则拆解，必须用 GPT-4 级别的模型。

## 做法：在 OpenClaw Router 上实现模型调度

OpenClaw 的 `router` 配置块支持基于条件表达式（`when`）分发到不同 `model` 后端。下面直接贴配置骨架（`.openclaw/config.yaml`）：

```yaml
models:
  - id: gpt4o-mini
    provider: openai
    model: gpt-4o-mini
    api_key: ${OPENAI_API_KEY}
  - id: local-qwen2.5
    provider: openai-compatible
    base_url: http://127.0.0.1:8000/v1
    model: qwen2.5:7b-instruct
    api_key: "na"

router:
  default_model: gpt4o-mini
  rules:
    - name: sensitive_local_only
      when: "context.user.is_internal is false and context.input.contains_pii == true"
      model: local-qwen2.5
      allow_fallback: false   # 禁止降级到云端

    - name: cheap_extraction
      when: "context.task.type in ['summary', 'keyword_extract', 'log_parse']"
      model: local-qwen2.5

    - name: hard_reasoning
      when: "context.task.complexity == 'high' or context.input.length > 3000"
      model: gpt4o-mini

    - name: catch_all
      model: gpt4o-mini
```

在 Python 脚本中，你只需要设置 `context`：

```python
ctx = openclaw.get_context()
ctx.task.type = "summary"
ctx.input.length = 1200
openclaw.run_agent("log_reducer", input="...", context=ctx)
```

Router 会按顺序匹配第一条 `when` 为真的规则。`allow_fallback: false` 表示本地模型出错时**不自动切到 GPT**，这对隐私防护很关键。

本地模型通过 Ollama 或 vLLM 暴露 OpenAI 兼容 API，OpenClaw 可以无感切换。

## 踩坑点

### 1. 本地模型输出格式不稳定
Router 把任务送给本地模型后，返回的 JSON 可能多一个感叹号、丢失右括号。如果下游是 MCP 工具调用，直接 500。  
- **解法**：在 local 模型规则中**强制使用 Grammar 约束**（如 llama.cpp 的 GBNF）或要求 vLLM 的 `guided_json`。若无法使用，则在 OpenClaw 的 post-processor 中用正则提取并做一次 `json.loads` 重试，失败时返回携带错误信息的标准化结构。

### 2. 提示词不兼容
GPT 系统提示里写的“请以 JSON 输出”在本地小模型上不一定遵守。你需要为本地模型单独维护一套简洁、强指令的提示词模板。  
- **建议**：在 `models` 定义中加 `system_prompt_override`，或在 Router 规则里动态选择 prompt template。

### 3. 并发与 GPU 内存
本地模型的并发推理常常是串行队列，当多个流水线同时触发 `cheap_extraction` 规则，延迟可能恶化到不可用。  
- **建议**：对走本地模型的 node 设置 `max_concurrency` 并加入任务队列监控；或者使用多实例 vLLM 前端。

### 4. 上下文长度误判
`context.input.length` 很容易计算，但有些任务短文本却需要极强推理。纯长度规则会错误路由到本地模型导致质量崩塌。  
- **对策**：引入一个快速分类器（甚至用一个很小的 BERT 模型）对复杂度打标签，成本极低，可作为 `when` 条件。

## 可复用建议

1. **建立任务 - 模型矩阵**  
   整理出 10~20 个典型自动化任务，给每个任务打上“复杂度/敏感度/吞吐量”标签，先全部用 GPT 跑 100 次获得 baseline 准确率，再逐步迁移到本地模型并对比错误率。定义可接受的退化阈值（例如 1%），稳定后上线路由规则。

2. **始终有 fallback 路径，但区分隐私**  
   非敏感任务允许 fallback 到 GPT 保证可用性；敏感任务必须 `allow_fallback: false`。否则某天 Ollama 挂掉，内部邮件就飞出墙了。

3. **监控路由分布和成本**  
   在 OpenClaw 的 telemetry 中为每个 `model` 增加标签，统计每日调用量、延迟分布、token 消耗。一旦看到本地模型错误率突增，快速切回 GPT 并排查。

4. **不要忽视本地模型的热身**  
   GPU 模型加载可能需要数秒，定时预热请求（比如每分钟发一个空总结）可避免冷启动导致的超时。

5. **用 feature flag 控制路由**  
   将路由规则配置外部化，通过配置中心动态调整，而不用改代码重启。这对线上调优非常有用。

## 总结

多模型路由的本质是把模型当作一种可调度资源。在 OpenClaw 这样的 Agent 框架里，Router 的规则引擎让我们能以工程化的方式管理“什么任务值得用 GPT、什么任务交给本地模型”。通过清晰定义任务特征、做好格式兼容与隐私硬隔离，你可以在几乎不损失能力的前提下，将 API 成本降低 60% 以上，同时保证核心数据不出网。这比你想象中更容易落地，值得每一个有实际工作流的团队立刻尝试。

---

