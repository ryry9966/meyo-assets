---
title: OpenClaw 多模型路由实战：什么时候该切 GPT，什么时候必须上本地模型
feedId: 32493
source: 综合讨论
publishedAt: 2026-08-11
---

# OpenClaw 多模型路由实战：什么时候该切 GPT，什么时候必须上本地模型

## 背景：一个 Agent 跑着跑着就破产了

刚开始用 OpenClaw 搭自动化工作流时，我习惯把所有任务都甩给 GPT-4。原因很简单：它理解力强，prompt 容错率高，搭原型飞快。但很快问题就来了——一个简单的网页内容清洗 Agent，每天要处理几百条 HTML 片段，月底的 token 账单让我开始怀疑人生。更麻烦的是，某些涉及内部 API 密钥或未脱敏数据的任务，直接把原始文本送到 OpenAI 的 API 上，安全评审直接被打回。

这就是典型的单一模型滥用问题。OpenClaw 本身支持多模型后端切换，配合简单的条件路由，能在成本、延迟、安全之间找到平衡。核心难点不是“能不能用”，而是“什么时候该用哪个”。

## 问题定义：三类任务的模型需求分化

我把日常 Agent 任务分成三类，每一类的模型需求完全不同：

1. **高理解力任务**：意图解析、复杂指令分解、开放式问答、生成带格式的报告。这类任务 GPT-4 / Claude 3.5 是刚需，本地 7B 模型基本不可用。
2. **结构化提取与清洗**：从 HTML、JSON 或日志里抽取固定字段，或者做格式标准化。这类任务用大模型是杀鸡用牛刀，本地小模型（如 Qwen2.5-7B-Instruct）配合严格 JSON schema 完全够用。
3. **敏感数据/离线任务**：需要处理内部文档、密钥、用户 PII，或者跑在无网环境里。这种场景本地模型不是可选项，是硬约束。

问题在于，同一 Agent 的不同步骤经常横跨这三类。比如一个“监控告警→分析日志→生成报告”的流程：日志清洗可以用本地模型，根因分析需要强推理，最终报告又需要 GPT 的排版和总结能力。所以需要在一个 OpenClaw 的 workflow 里实现动态路由。

## 做法：基于任务标签的模型路由

OpenClaw 本身不提供内置路由，但它的 tool 和 pipeline 架构允许你在定义步骤时，通过 env 或 executor 指定不同的模型后端。我的做法是把模型选择抽象成一个“模型网关”，在工作流定义里用标签声明任务类型，运行时再映射到具体模型。

### 步骤 1：准备两个模型后端

在 OpenClaw 的模型配置中，至少定义两个 provider：

```yaml
# openclaw.yaml
models:
  - id: gpt-4o
    provider: openai
    model: gpt-4o-2024-08-06
    api_key: ${OPENAI_API_KEY}
  - id: local-qwen
    provider: ollama
    model: qwen2.5:7b-instruct-q8_0
    base_url: http://localhost:11434/v1
```

本地模型我用的是 Ollama 部署的 Qwen2.5-7B，8-bit 量化，16G 内存的机器上推理延迟在 200ms 以内，足够用于流式处理。

### 步骤 2：在工作流里打任务标签

在 OpenClaw 的 workflow 定义中，给每个需要使用模型的 step 加上一个 `task_profile` 字段，比如：

```yaml
steps:
  - id: parse_alert
    task_profile: extraction
    prompt_template: |
      Extract alert name, severity, and host from:
      {{ input.alert_text }}
    output_format: json
  - id: root_cause_analysis
    task_profile: reasoning
    prompt_template: |
      ...
  - id: generate_report
    task_profile: generation
```

`task_profile` 可以自定义为 `extraction`、`reasoning`、`generation`、`offline` 等。这只是一个约定，实际运行时我会在 pipeline executor 里读取。

### 步骤 3：实现简单的路由执行器

我写了一个轻量 wrapper，在 step 执行前根据 `task_profile` 选择模型：

```python
MODEL_MAP = {
    "extraction": "local-qwen",
    "offline": "local-qwen",
    "reasoning": "gpt-4o",
    "generation": "gpt-4o",
}

def resolve_model(step):
    profile = step.get("task_profile", "reasoning")
    if os.environ.get("FORCE_LOCAL"):
        return "local-qwen"
    return MODEL_MAP.get(profile, "gpt-4o")
```

然后在 `run_step` 里调用 OpenClaw 的模型接口时，传入对应的 model_id。如果你用的是 OpenClaw 的 Python SDK，可以这样：

```python
result = openclaw.run(
    model=resolve_model(step),
    messages=[...],
    response_format=step.get("output_format")
)
```

对于 `extraction` 类任务，强制使用 `response_format={"type": "json_object"}` 或 tool calling 方式，能大幅提高本地小模型的输出稳定性。

## 踩坑点：本地模型不是免费的午餐

### 1. JSON 模式下的幻觉

Qwen2.5-7B 在 JSON 约束下偶尔会产生多余的 key 或遗漏字段。解决方法：
- 始终用 `json_schema` 生成器提前验证；
- 在 prompt 里给出一个具体的 JSON 示例，比单纯描述字段可靠得多；
- 如果某个字段抽取失败，设置 fallback 值并标记 `extraction_quality` 为 low，后续步骤可据此决定是否走人工审核或切回 GPT 重试。

### 2. Prompt 兼容性

GPT 优化过的 prompt 直接搬到小模型上大概率翻车。本地模型更喜欢明确、无歧义的指令，且对 system/user 角色分界没那么敏感。我建议为 `extraction` 类任务维护一套独立的小模型专用 prompt 模板，尽可能减少依赖推理的部分。

### 3. 并发与资源管理

Ollama 默认并行请求数有限，如果 Agent 并发量上来，本地模型很容易成为瓶颈。我现在的做法是对 `extraction` 任务做批量聚合：将多条待处理文本拼成一个 batch prompt，让模型一次返回 JSON 数组，吞吐量能提升 3-5 倍。

### 4. 安全边界

即使是本地模型，也不代表完全安全。如果你用本地模型处理未脱敏数据，要确保模型加载在内存里，且不记录到任何云日志中。另外，OpenClaw 本身的执行日志会记录 prompt 和输出，记得关闭对应 step 的日志持久化或者做脱敏处理。

## 可复用建议

经过几个项目的试错，我总结了一套判断准则，可以直接拿去用：

- **你愿意为这个任务等 2 秒以上吗？** 如果不能（比如实时告警拆分），优先本地模型。
- **这个任务的输出格式是否严格固定？** 如果是，先用本地模型 + JSON 模式做一个 baseline，质量不够再切 GPT。
- **prompt 里是否包含内部 URL、密钥或用户数据？** 哪怕只是一条，也必须走本地模型。
- **任务是否需要跨文档推理或复杂多跳？** 这类任务暂时别省，GPT-4 带来的稳定性远超节省的费用。
- **离线或 air-gap 环境**：完全没有选择，直接构建本地模型镜像。

另外，做个模型选择矩阵贴在团队 Wiki 里，减少每次的沟通成本：

| 任务类型 | 首选模型 | 备选模型（成本/离线） | 输出格式要求 |
| --- | --- | --- | --- |
| 意图识别、指令分解 | GPT-4o | Claude 3.5 | text |
| 结构化抽取 | local-qwen | GPT-4o-mini | JSON |
| 日志分析 | local-qwen | GPT-4o | JSON / text |
| 报告生成 | GPT-4o | local-qwen (草稿) | Markdown |
| 代码生成/审查 | GPT-4o | local-code-llm | code block |

## 总结

多模型路由不是“省钱”那么简单，它实际上是在重构 Agent 的可靠性模型。把那些重复、机械、敏感的任务从大模型身上剥离，不仅降低成本，还能让 GPT 专注于它真正擅长的推理和生成部分，避免被琐碎任务污染上下文窗口。

OpenClaw 的灵活性允许你几乎以零侵入的方式实现这种路由，但前提是你要清楚地定义任务的语义级别，并愿意为每类任务维护专属 prompt 和校验逻辑。初期搭建会多花一两天，但在第一个账单周期结束的时候，你就会觉得值了。

如果你同样在搞这种混合模型架构，欢迎在仓库里分享你的 task_profile 定义——社区里已经有好几个有趣的实践，比如把 embedding 任务也纳入路由、或者用质量置信度做回退切换，这些都是下一步可以探索的方向。

---

