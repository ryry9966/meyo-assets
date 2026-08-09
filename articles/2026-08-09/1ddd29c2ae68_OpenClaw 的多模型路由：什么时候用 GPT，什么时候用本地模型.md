---
title: OpenClaw 的多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 32234
source: 综合讨论
publishedAt: 2026-08-09
---

# OpenClaw 的多模型路由：什么时候用 GPT，什么时候用本地模型

## 背景

在 OpenClaw 的自动化流水线里，模型不再是一个固定的后端，而是可以根据任务动态调度的资源。常见的组合是：OpenAI 的 GPT 系列（GPT-4o、GPT-4o-mini）提供强推理能力，本地模型（通过 Ollama 部署的 Llama 3、Qwen2.5 等）提供低延迟、零数据外泄、无限调用的补充。

但很快会碰到一个工程问题：**到底什么时候该用 GPT，什么时候该走本地模型？** 如果在所有 Agent 任务上都无脑走 GPT，月底账单会教你做人；如果一刀切全用本地 7B/8B 模型，遇到复杂推理、多步工具调用，任务就会静默失败，反复重试反而更慢。

这篇帖子不讨论“哪个模型更好”，而是整理一套在多模型可用的前提下，能直接用在 OpenClaw/MCP 插件链路上的路由决策方案与避坑记录。

## 问题拆解

常见的 OpenClaw 工作流里，Agent 会经过多个步骤：意图分类、工具调用、代码生成、文本总结、邮件润色、数据提取等。并不是每个环节都需要 GPT-4 级别的推理。典型矛盾包括：

- **成本**：批量处理邮件摘要，一天上万条，用 GPT-4 成本暴增，而本地 7B 模型足以胜任。
- **隐私**：处理内部 HR 文档或密钥相关日志时，数据不能离开本机，只能用本地模型。
- **质量**：需要生成带复杂逻辑的 SQL 或 Python 脚本时，本地小模型容易产生幻觉或错误语法，必须上 GPT。
- **延迟**：实时交互场景下，本地模型 token 生成速度可能比 API 快，但首 token 延迟取决于硬件。

因此，需要一套能根据任务特征、环境上下文动态选择模型的路由机制，而不是写死在某一个配置里。

## 做法与步骤

我们最终采用了“任务标签 + 成本/隐私阈值”的显式路由，而不是让模型自己去选（那样本身也要消耗 token）。整体结构如下：

**1. 定义模型池与能力标签**

在 OpenClaw 的 `config/models.yaml` 或插件配置中，明确每个模型的“能力画像”：

```yaml
models:
  - name: gpt-4o
    type: api
    strengths: [complex_reasoning, code_generation, tool_calling]
    cost_per_1k_tokens: 0.015
    max_tokens: 128000
  - name: gpt-4o-mini
    type: api
    strengths: [simple_qa, summarization, translation]
    cost_per_1k_tokens: 0.0006
    max_tokens: 128000
  - name: local-qwen2.5-7b
    type: local
    strengths: [summarization, extraction, formatting, sensitive_data]
    cost_per_1k_tokens: 0
    max_tokens: 32768
```

**2. 对任务打标签**

通过 MCP 服务或 Agent 内部预处理，给每个任务打上标签。例如：

- 用户输入 → 意图分类 → `tag: summarization`  
- 系统日志解析 → `tag: extraction`  
- 用户要求生成 Ansible playbook → `tag: code_generation`  
- 处理含员工 ID 的文本 → `tag: sensitive_data`

可以用极低成本模型（甚至本地 1B 模型）做分类，这部分开销基本可忽略。

**3. 实现路由函数**

在 OpenClaw 的插件或 Agent 调度层实现一个简单的路由器：

```python
def route_model(task_tag: str, context: dict) -> str:
    # 强隐私要求 -> 强制本地
    if task_tag == "sensitive_data" or context.get("local_only"):
        return "local-qwen2.5-7b"
    # 复杂推理或代码生成 -> GPT-4
    if task_tag in ["complex_reasoning", "code_generation"]:
        return "gpt-4o"
    # 简单任务 -> gpt-4o-mini 或本地，取决于上下文长度与预算
    if task_tag in ["summarization", "extraction", "formatting"]:
        if context.get("input_length", 0) > 20000:
            return "gpt-4o-mini"  # 超长上下文本地模型可能超出窗口
        if context.get("monthly_budget_exhausted"):
            return "local-qwen2.5-7b"
        return "gpt-4o-mini"
    # 兜底
    return "gpt-4o-mini"
```

触发任务时，Agent 从配置中获取当前可用模型列表，根据标签选择模型名，然后动态设置 `model` 参数再执行推理。OpenClaw 的插件体系允许在执行 Tool/Chain 之前插入这样一个前置步骤。

**4. 连接 MCP 工具时的模型选择**

当 Agent 通过 MCP 下发工具调用（例如文件读取、数据库查询），工具返回的结果可能需要再次推理。此时仍沿用路由逻辑：如果工具结果是大段原始日志，用本地模型做提取；如果是简短错误码，用 GPT-4o-mini 生成解决建议。

## 踩坑点

- **本地模型输出格式不稳定**：本地 7B 模型在结构化输出（如 JSON）上远不如 GPT。必须加后处理或用 guided generation（如 llama.cpp grammar），否则下游工具解析失败。我们的做法是：对格式要求严格的路径，即使内容简单，也优先用 gpt-4o-mini。
- **本地模型首 token 延迟**：首次加载模型可能耗时数秒，不适合实时交互。建议让 Ollama 常驻，或使用 `keep_alive` 避免频繁卸载。
- **多模型会话状态混乱**：如果同一个对话历史里混用不同模型，容易因为上下文格式差异导致生成异常。我们在路由边界上会清空或重置 system prompt，而不是在同一条历史里来回切换。
- **成本监控缺失**：自以为走了很多本地模型，但一个小 bug 把所有流量打到 GPT-4o 上。必须每个任务记录模型名与 token 数，否则算账全靠猜。

## 可复用建议

- **从小处开始**：先只在“隐私”和“大批量文本摘要”两条路径引入本地模型，验证稳定性后再扩展。
- **建立成本/质量日志**：记录每次调用的模型、token 消耗、错误重试次数，一周后就能看出哪些标签可以用更便宜的模型替代。
- **抽象路由为独立 MCP 工具**：把路由逻辑封装成一个 MCP server，让不同 Agent 都能调用，避免散落在多个插件里。
- **本地模型做预处理，远程模型做决策**：让本地模型提取、格式化、过滤，GPT 做最终判断或生成，这种组合在日志分析和数据清洗中很有效。

## 总结

多模型路由不是简单的“哪个便宜用哪个”，而是要把任务属性、数据敏感度、上下文长度、预算余量都考虑进去。在 OpenClaw 体系里，通过显式的任务标签加轻量路由函数，可以在几乎不增加工程复杂度的情况下，把大量低价值、高频率的推理转移给本地模型，把宝贵的 GPT 配额留给真正需要强推理的环节。这种做法实测可以让整体 API 成本降低 40–60%，同时不牺牲自动化流水线的可靠性。

---

