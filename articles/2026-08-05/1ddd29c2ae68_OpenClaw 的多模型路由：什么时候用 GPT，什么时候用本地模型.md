---
title: OpenClaw 的多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 31764
source: 综合讨论
publishedAt: 2026-08-05
---

## 为什么需要多模型路由

在 OpenClaw 的 Agent 自动化里，模型已经不只是“调一个 API”这么简单。一方面 GPT-4o、Claude 3.5 Sonnet 等云端模型在复杂推理、长上下文、多模态任务上依然优势明显；另一方面本地部署的量化模型（如 Qwen2.5-14B、DeepSeek-V2-Lite 通过 Ollama/Llama.cpp）在简单文本处理、低延迟、数据不出域等场景下性价比极高。完全云端成本不可控，完全本地能力有天花板，纯粹靠人判断又不自动化——这就是多模型路由要解决的问题。

## 什么时候用 GPT，什么时候用本地模型？

这是一个工程决策，而不是“哪个模型强就用哪个”。在 OpenClaw 的实践中，我通常用以下几个维度做判断：

- **任务复杂度**：简单翻译、摘要、关键词提取、意图分类等，14B~32B 的本地模型已足够可靠；涉及多步推理、跨文档逻辑、复杂代码生成或调试的任务，交给 GPT-4o 或 Claude。
- **隐私与合规**：包含客户 PII、内部文档、合同条款的内容，默认走本地模型，必要时甚至可以关闭云端 fallback。
- **延迟要求**：本地模型在 warm 状态下首 token 延迟可低于 200ms，适合实时交互的 Agent 插件；云端模型网络 + 推理延迟通常 >1s，只适合对实时性不敏感的阶段。
- **成本**：如果某类任务日均调用量上万次，全部走 GPT-4o-mini 仍会产生显著费用。把其中 70% 的简单请求切到本地模型，成本直接降到几乎为 0。
- **结构化输出稳定性**：需要严格 JSON/function calling 的场景，本地模型目前仍有较高概率产生格式错误，云端模型在此更稳定。可以设计“云端主输出，本地后处理”的混合模式。

## 在 OpenClaw 里怎么实现路由

OpenClaw 的模型路由可以通过 `providers` + `router` 配置完成，不需要改动业务逻辑代码。以下是一个可复用的配置示意：

```yaml
providers:
  - id: gpt4o
    type: openai
    model: gpt-4o
    api_key: ${OPENAI_API_KEY}
    max_tokens: 4096
  - id: local-qwen
    type: ollama
    model: qwen2.5:14b
    base_url: http://localhost:11434
    keep_alive: 5m    # 保持模型常驻，避免冷启动

router:
  rules:
    - name: high_security
      when: request_context.security_level == 'high'
      use: local-qwen
      fallback: none   # 禁止降级到云端

    - name: complex_tasks
      when: >
        prompt_text matches "(analyze|debug|refactor|multi-step)" 
        and estimated_tokens > 800
      use: gpt4o

    - name: function_calling
      when: request_context.tool_choice is not None
      use: gpt4o

    - default: local-qwen
```

路由规则可以基于多种信号：prompt 文本正则、上下文标签（来自 MCP 插件或前置步骤）、token 估算值、甚至时间窗口（例如工作时间用云端，非峰值用本地）。OpenClaw 的 Router 还支持自定义 `classifier` 插件，可以接一个极小的 BERT 模型做意图分类，然后把分类结果作为路由条件——这对于复杂路由逻辑非常有用。

## 三个容易踩的坑

**1. 本地模型上下文窗口不够用**  
本地模型默认 `n_ctx` 可能只有 4096，一旦 Agent 串联了长文档或 MCP 工具返回大量文本，很容易触发截断，导致输出丢失关键信息。务必在 Ollama 启动参数中调整 `num_ctx`，并做好上下文长度监控，超过阈值强制路由到云端或做摘要压缩。

**2. Function Calling / JSON 模式不稳定**  
本地模型即使在 prompt 里要求输出 JSON，也经常多出一个解释性前缀或缺少闭合括号。不要假设本地模型能直接返回合法 JSON。可以在 OpenClaw 的 output parser 里加入正则提取、重试、schema 校验，或者干脆让本地模型只输出自由文本，再由一个极小的正则引擎做结构化。

**3. 本地模型冷启动导致超时**  
容器化部署或按需加载时，首次请求可能耗时 10~30 秒，超出了 Agent 的默认超时。解决方案：配置 `keep_alive` 让模型常驻内存，或使用 `warmup` 插件在 Agent 启动后立即发一个空白请求预热。

## 可复用的工程建议

- **从日志里找路由信号**：先跑一段时间纯云端，分析 prompt 的模式，提取高频简单任务的特征词，沉淀为路由规则。
- **路由规则配上 fallback**：本地模型超时或格式错误时，能自动兜底到云端（隐私场景除外），同时记录 fallback 事件用于事后调优。
- **成本监控不可缺**：给每个 provider 打标签，在 OpenClaw 的 trace 里统计各模型调用次数、tokens 和延迟，方便随时调整路由权重。
- **MCP 工具链统一上下文**：如果使用了 MCP 插件提供外部数据，可以在 MCP 返回里标注敏感等级，传递给 Router 做决策，避免敏感数据误入云端。

## 总结

多模型路由不是非此即彼的选择，而是一套持续的流量调度优化。核心原则很简单：能用本地解决的绝不走云端，复杂的、要求高准确率和结构化输出的果断用 GPT，但永远给本地模型一个兜底并监控它的表现。在 OpenClaw 里花半小时配置好路由规则，未来省下的 API 费用和排查隐私问题的时间会远超过投入。

---

