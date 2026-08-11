---
title: OpenClaw 多模型路由：区分 GPT 与本地模型的使用边界
feedId: 32555
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在 OpenClaw 这类 Agent/MCP 框架中，模型不再只是对话的终端，而是被编排进插件调用、工具链和多步推理的流程里。一个常见的工程冲动是“全部用最强的模型”，但成本、延迟和数据驻留会很快教我们做人。另一极端是咬牙把 Llama、Qwen 等本地模型塞进每一个环节，结果长上下文推理或复杂代码生成频繁翻车。

折中是——**多模型路由**：在同一个 Agent 上下文中，根据任务动态选择云端大模型（GPT-4o、Claude 等）还是本地模型（通过 Ollama、llama.cpp 或 vLLM 部署）。OpenClaw 提供了多后端支持和灵活的 hook 机制，让这件事变得可落地。本文将分享我们在实践中沉淀出的路由策略、配置方式与踩坑记录。

## 问题拆解

核心需要回答两个问题：

1. **什么时候该用 GPT？**
2. **什么时候该用本地模型？**

单纯靠经验规则（如“简单问题用本地，难问题用 GPT”）并不可靠，因为难易边界很主观。我们需要一套可量化的分类维度：

- **任务复杂度**：是否需要多步推理、长上下文理解、代码生成或复杂结构化输出？
- **数据敏感度**：输入中是否包含用户隐私、内部文档、密钥等不应离开本机的信息？
- **延迟要求**：能否忍受云端 API 的网络往返时间？某些高频工具调用对延迟极度敏感。
- **上下文长度**：本地小模型在 8k 以上上下文时可能出现注意力弥散，而 GPT-4-turbo 可以稳定处理 128k。
- **成本控制**：每日预算有限时，需要把高成本调用留给高价值任务。
- **功能依赖**：是否需要 OpenAI 的 function calling 严格模式、JSON mode 等本地模型尚不稳定的能力。

根据这些维度，我们为常见任务建立一个**路由决策矩阵**，示例如下：

| 任务类型               | 推荐模型      | 决策依据                         |
| ---------------------- | ------------- | -------------------------------- |
| 简单分类、意图识别     | 本地小模型    | 低复杂度、低时延、成本几乎为零   |
| 数据清洗/格式化        | 本地模型      | 数据敏感性高、上下文短           |
| RAG 检索后的答案生成   | 本地/云端混合 | 若无敏感数据，可云；有，则本地   |
| 多步代码生成与调试     | GPT-4o        | 强推理、长上下文、结构化输出     |
| 复杂 SQL/API 编排     | GPT-4o        | 需要准确的 function calling      |
| 用户对话首轮分类       | 本地模型      | 低延迟，快速路由至后续专业 handler |
| 敏感文档摘要           | 本地模型      | 数据绝不能离开本机               |
| 多轮长上下文对话管理   | GPT-4-turbo   | 需要记忆和注意力稳定             |

## 在 OpenClaw 中的实现步骤

### 1. 准备多模型后端

确保 OpenClaw 能够同时访问云端 API 和本地推理服务。本示例中，我们使用 Ollama 提供本地模型（`qwen2.5:7b`），云端使用 Azure OpenAI GPT-4o。

在 `openclaw.yaml` 中定义两个后端：

```yaml
model_backends:
  gpt4:
    provider: openai
    model: gpt-4o
    api_key: ${OPENAI_API_KEY}
    endpoint: https://xxx.openai.azure.com/
  local_qwen:
    provider: ollama
    model: qwen2.5:7b
    base_url: http://localhost:11434
```

### 2. 设计路由模型（Router）

路由本身也可以由一个轻量模型完成。我们让 OpenClaw 在每次接收到用户请求时，先调用一个“分类器 tool”，该 tool 使用本地模型（成本低、延迟低）判断复杂度、敏感度并返回目标模型名称。伪代码如下：

```python
@tool
def decide_model(user_input: str) -> str:
    prompt = f"""Analyze the following user request and decide which model to use:
- return "local" if the request is simple, no complex reasoning or code generation, and does not require long context.
- return "gpt" if the request involves multi-step reasoning, coding, sensitive data must stay local anyway? Actually if sensitive, force local.
- If any sensitive data is mentioned, return "local" regardless.
Request: {user_input}
Decision (local/gpt):"""
    # 用本地小模型快速推理
    response = ollama.chat(model="qwen2.5:7b", messages=[{"role": "user", "content": prompt}])
    decision = response['message']['content'].strip().lower()
    return "local" if "local" in decision else "gpt"
```

将这个 tool 注册到 OpenClaw 的 `pre_process` hook 中，用于设置 `session.model`。

### 3. 配置 Hook 实现动态切换

OpenClaw 支持在会话生命周期中注入钩子。我们在 `hooks` 中定义：

```yaml
hooks:
  pre_process:
    - type: tool
      tool: decide_model
      output_mapping:
        target_model: "{{ result }}"
  session_start:
    - type: set_model
      model_from: "{{ target_model }}"
```

这样，每次新会话或新消息到来，`decide_model` 会先执行，并将结果传回，随后 OpenClaw 使用对应的模型进行后续处理。对于实时性要求极高的场景，甚至可以跳过分类器，直接基于简单关键词规则（如检测到“代码”、“SQL”等字眼）设置模型，但通常分类器更稳健。

### 4. 处理敏感数据回路

当 `decide_model` 识别到输入包含 API 密钥、内部文档片段时，强制返回 `local`。同时我们会在 local 模型链里关闭任何向云端发送数据的 tool（比如 search、web fetch），避免间接泄露。这需要在 OpenClaw 的 `tool_allowlist` 里按模型动态配置，可借助 `per_model_config` 实现。

### 5. 回退机制

本地模型有时会因为幻觉或能力不足返回低质量结果。我们可以设置置信度回退：在执行关键 tool（如代码执行）之前，让本地模型输出自评置信度，若低于阈值则转投 GPT。这需要额外的一步校验 tool，增加约 0.3 秒延迟，但换取可靠性。

```python
@tool
def confidence_check(local_output: str) -> bool:
    prompt = f"Rate your confidence in this output (0-10). Only return the number:\n{local_output}"
    resp = ollama.chat(model="qwen2.5:7b", ...)
    score = int(resp.strip())
    return score >= 7
```

## 踩坑点

1. **分类器本身不准**：用本地 7B 模型做意图分类有时会高估用户请求的复杂度，导致频繁回退到 GPT，成本依然高。需要针对业务微调分类 prompt，或收集一批真实 case 做 few-shot。
2. **本地模型冷启动**：Ollama 首次加载模型需要数秒，这对于对话首条消息不可接受。必须设置常驻内存（如在启动脚本中预先调用一次推理），并保持连接池。
3. **上下文格式不兼容**：GPT 使用 ChatML 或系统提示，而本地模型可能要求不同的模板。来回切换模型时，若 session 中已有多轮对话，需要重构消息格式。OpenClaw 尚未内置自动转换，我们在路由切换时清空历史或仅保留压缩摘要。
4. **function calling 范式差异**：本地模型对 OpenAI 风格的 tool calling 支持参差不齐。若路由到本地模型却要进行结构化 tool 调用，容易失败。因此我们将需要严格 function calling 的任务固定留给 GPT，本地模型仅负责自然语言输出或简单的 JSON 模式。
5. **日志监控成本**：多模型路由使得每次请求成本不透明。务必在日志中印出模型选择和 token 用量，方便后续优化。

## 可复用建议

- **抽象出 ModelRouter 模块**：将决策矩阵、分类器、回退逻辑封装成独立插件，这样在不同 OpenClaw 项目间复用，只需调整阈值和模型名。
- **用规则兜底**：分类器虽灵活，但关键路径（如包含明显敏感词）直接用简单正则拦截，避免模型判断失误。
- **建立基准测试**：针对你的典型任务集，定期评测本地模型和 GPT 的边界在哪里，调整路由矩阵。比如发现 `qwen2.5:7b` 在 4k 上下文以内的 SQL 生成准确率可接受，就扩宽路由范围。
- **成本可视化**：在 Dashboard（如果有）或日志中统计“本地节省成本”指标，这能帮助团队认可多模型策略的价值。

## 总结

在 OpenClaw 中实现多模型路由，本质是将“用能力换成本”和“用本地换隐私”这两个天平数字化、自动化。没有银弹，但通过决策矩阵 + 轻量分类器 + 回退机制，我们能在不明显牺牲体验的前提下，把高价值任务留给 GPT，日常杂务、敏感任务留给本地模型。实际运行三个月下来，我们在一个代码助手场景中，云端调用量降低了 62%，而任务成功率基本持平。

把路由当成一个需要持续迭代的组件，而不是一次性配置，才能跟上模型能力的进化和业务的变化。

---

