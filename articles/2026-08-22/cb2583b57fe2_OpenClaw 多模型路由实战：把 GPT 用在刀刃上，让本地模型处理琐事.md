---
title: OpenClaw 多模型路由实战：把 GPT 用在刀刃上，让本地模型处理琐事
feedId: 34219
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 上跑了几个月的自动化任务，包括网页摘要、数据清洗、意图识别和定时报告。一开始所有请求都直接走 GPT，账单增长很快，而且部分任务涉及内部数据，不希望上云。后来在本地用 Ollama 部署了 7B 模型，但直接替换又遇到复杂推理乏力、输出格式不稳定等问题。于是开始做多模型路由，把任务按特征分流给 GPT 或本地模型。

## 问题

OpenClaw 默认使用单一模型配置，但实际任务对模型能力、延迟、隐私、成本的需求差异很大。比如意图识别、关键词抽取这类简单任务，用 7B 本地模型完全够用；而长文摘要、复杂推理、严格 JSON 输出则需要 GPT。如何在 OpenClaw 中实现自动路由，同时保证失败回退和可观测性，是这篇内容要解决的问题。

## 做法/步骤

### 1. 任务画像与边界划分

先把日常的 OpenClaw 自动化任务拆成三类：

- **简单抽取/分类**：意图识别、情感分析、关键词提取、数据脱敏
- **生成式长文本**：摘要生成、报告撰写、邮件草拟
- **严格结构化输出**：需要稳定 JSON 或 YAML 的步骤，比如 MCP 工具调用参数生成

用本地模型分别测试这三类任务，记录失败率、输出格式错误率和延迟。实测中，7B 量化模型在简单抽取上准确率接近 GPT，但生成长文本和严格 JSON 的失败率明显偏高。

### 2. 准备本地模型

使用 Ollama 部署 `qwen2.5:7b-instruct-q4_K_M`，4-bit 量化，显存占用约 5GB，在 24GB 显卡上可以流畅运行。该模型支持 OpenAI 兼容 API，方便 OpenClaw 接入。

### 3. OpenClaw 多 Provider 配置

在 OpenClaw 的模型配置中增加两个 provider，分别指向 OpenAI 和本地 Ollama，设置模型别名：

```yaml
providers:
  - name: openai
    base_url: https://api.openai.com/v1
    api_key: ${OPENAI_API_KEY}
    models: [gpt-4o-mini, gpt-4o]
  - name: ollama
    base_url: http://localhost:11434/v1
    models: [qwen2.5:7b-instruct-q4_K_M]
aliases:
  gpt: openai/gpt-4o-mini
  local: ollama/qwen2.5:7b-instruct-q4_K_M
```

### 4. 实现路由中间件

利用 OpenClaw 的插件/钩子机制，或者在任务脚本入口调用一个独立的路由函数。我采用轻量规则 + 白名单的方式，避免复杂的 NLU 分类：

```python
def route_task(task):
    if task.type in ["summarize", "reasoning", "long_form"]:
        return "gpt"
    if task.type in ["classify", "extract", "sentiment"]:
        return "local"
    if has_sensitive_data(task.text):  # 身份证号、API key 等
        return "local"
    if len(task.text) > 8000:  # 长上下文任务
        return "gpt"
    return "gpt"  # 默认走 GPT
```

路由函数返回别名，再传给 OpenClaw 的模型选择参数。

### 5. 加入回退与日志

- **本地模型回退**：如果本地模型超时、返回格式解析失败，自动降级到 GPT。
- **GPT 回退**：如果是简单任务且 GPT 请求失败，尝试本地模型兜底。
- **日志记录**：每次请求记录模型名、延迟、token 消耗、任务类型，方便后续调整阈值。

## 踩坑点

### 本地模型输出格式不稳定

让 7B 模型输出 JSON 时，必须在 prompt 里给出明确 schema，例如：

```text
只输出 JSON，不要有任何解释，格式为 {"category": "xxx", "confidence": 0.9}
```

并在 OpenClaw 中做 JSON 解析校验，失败则重试一次，再失败才回退到 GPT。

### 上下文长度截断

7B 模型默认上下文窗口可能只有 4k 或 8k，长文档任务必须路由给 GPT，否则内容被静默截断，结果看起来正常但实际不完整。

### 并发显存冲突

本地模型推理时会占用 GPU，如果多个 OpenClaw 任务同时触发，容易导致显存不足或推理变慢。建议对本地 provider 设置并发限制（例如每次只允许 2 个请求）。

### 路由规则过于死板

早期用纯关键词路由，误判率高。后来改为“轻量规则 + 白名单”，并定期根据日志调整。不建议引入额外的分类模型做路由，会增加一次推理延迟，得不偿失。

### 版本耦合

OpenClaw 更新可能改变插件接口，将路由逻辑封装成独立脚本，通过环境变量或 CLI 参数传入，可以降低耦合。

## 可复用建议

1. **简单、高频、低敏感度任务优先本地**：意图识别、关键词提取、数据脱敏、文本分类。
2. **复杂推理、长文生成、严格格式输出、需要最新知识的任务优先 GPT**。
3. **路由决策集中在单一模块**，避免散落在多个脚本里，便于维护和统计。
4. **固定本地模型的 prompt 模板和输出格式**，减少解析失败。
5. **定期统计各任务类型的模型成功率和延迟**，调整路由表阈值。
6. **对本地模型单独限制并发**，避免 GPU 争抢导致整体变慢。

## 总结

多模型路由不是一次配置完就结束的事，而是一个持续调优的工程过程。在 OpenClaw 上通过多 provider 配置 + 一层轻量路由，可以把 GPT 用在真正需要的地方，同时保留本地模型的低成本和隐私优势。目前我的自动化任务中约 60% 的请求被本地模型接管，整体 token 费用明显下降，核心任务质量没有受到影响。关键在于清晰的任务边界、可靠的回退机制和足够的可观测性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e743a18fb96bcce5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a20857c1546be3a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/f2eff389b321198c.png)

