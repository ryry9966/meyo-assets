---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 33450
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

很多 OpenClaw/Agent 实践者在早期都习惯直接接云端大模型 API，省事，但问题也明显：费用不可控、请求限流、数据隐私敏感，以及断网或内网环境下无法使用。对于经常跑 MCP 工具、写插件、做自动化任务的用户来说，如果有一个本地可用的 LLM 推理后端，能显著降低测试成本，也能把敏感数据留在自己机器上。

但家用电脑跑大模型不是“下载模型文件”这么简单。显存、内存、量化方式、推理引擎、工具调用稳定性，都会直接影响 Agent 能不能用。

## 问题

家用电脑资源有限：多数人显卡显存 8GB 到 16GB，内存 16GB 到 32GB。未经量化的 7B 模型 FP16 权重就要 14GB，直接加载都困难。更麻烦的是，很多小模型在工具调用、结构化输出上表现不稳定，而 OpenClaw/Agent 场景恰恰依赖这些能力。如果只跑闲聊，能用；一旦接 MCP 工具、让模型决定调用哪个插件、传什么参数，问题就暴露了。

因此本地部署的目标不仅是“能跑”，还要“跑得稳、能接 Agent”。

## 做法/步骤

### 1. 硬件与模型匹配

先明确自己的硬件上限：

- 8GB 显存：优先 7B/8B 模型，量化 Q4_K_M 或 Q5_K_M。
- 16GB 显存：可以跑 13B/14B Q4，或 7B/8B 更高量化。
- 32GB 内存 + 无独显：可以跑 7B Q4，但 token 速度通常只有 3-8 tok/s，适合低频任务。

不要硬上 30B/70B，即使能加载，推理速度也会让 Agent 多轮调用变得不可用。

### 2. 选推理引擎

推荐三个方向：

- **Ollama**：安装简单，自带 OpenAI 兼容 API，适合快速接入 OpenClaw。
- **llama.cpp server**：更可控，适合需要自定义参数、grammar、KV cache 的场景。
- **LM Studio**：图形化操作，适合不想折腾命令行的用户。

如果目标是接 OpenClaw，优先选 Ollama 或 llama.cpp server，因为它们都能暴露 `/v1` 端点。

### 3. 模型选择

优先选支持 function calling / tool use 的 instruct 模型。实测在 OpenClaw 场景下比较稳的有：

- `Qwen2.5-7B-Instruct`
- `Mistral-7B-Instruct-v0.3`
- `Llama-3.1-8B-Instruct`
- `DeepSeek-Coder-V2-Lite-Instruct`

量化格式用 GGUF，Q4_K_M 是性价比首选，Q5_K_M 质量更好但略慢。

### 4. 部署与接入

以 Ollama 为例：

```bash
# 拉取模型
ollama pull qwen2.5:7b-instruct-q4_K_M

# 启动服务并暴露 OpenAI 兼容端点
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

然后在 OpenClaw 配置中把模型后端指向本地：

```yaml
base_url: http://localhost:11434/v1
model: qwen2.5:7b-instruct-q4_K_M
```

建议先跑一轮普通对话，再测试工具调用。可以用一个简单 MCP 工具（比如获取时间、查询文件）验证模型是否能正确输出工具名和参数。

### 5. 调优

把 temperature 调到 0.1-0.3，减少随机性。限制上下文长度，7B/8B 模型建议 `ctx_size` 设为 4096 或 8192，太高容易 OOM 或注意力分散。必要时用 JSON schema 或 grammar 约束输出。

## 踩坑点

1. **显存不足/OOM**：最常见。往往是上下文开太大，或者 KV cache 没量化。解决：降低 `num_ctx`，换 Q4 量化，或在 llama.cpp 中启用 `--kv-cache-dtype f16` 改为更低精度。

2. **CUDA/驱动不匹配**：Linux 上 llama.cpp 编译时没带 CUDA，或者 Windows 上显卡被 Ollama 识别为 CPU。需要确认引擎是否真正使用了 GPU，可以用 `nvidia-smi` 观察显存占用。

3. **工具调用不稳定**：小模型可能输出错误 JSON、调用不存在的工具，或者把工具名拼错。解决：选支持 tool calling 的模型；在系统提示里明确列出可用工具和参数格式；降低 temperature；必要时用 grammar 强制 JSON 输出。

4. **CPU 推理太慢**：7B Q4 在纯 CPU 上跑多轮 Agent 任务，等待时间可能超过一分钟。如果只是做简单文本处理或单次总结还能接受，复杂多步 Agent 建议还是用 GPU 或换更小模型。

5. **上下文截断**：OpenClaw 配置里如果没注意模型最大 context，长对话或 MCP 工具描述过多会导致请求报错。小模型对长工具列表的处理也不好，建议精简工具描述，或者按需加载工具。

## 可复用建议

- 从 `7B/8B Q4_K_M` 开始，先验证工具调用稳定性，再考虑换大模型。
- 用 Ollama 的 OpenAI 兼容模式接入 OpenClaw，减少适配成本。
- 给 Agent 写专用系统提示，明确输出格式、可用工具、错误处理方式，别让小模型自由发挥。
- 设置 `temperature=0.1-0.3`，限制 `num_predict`，防止模型跑飞。
- 记录工具调用失败 case，针对性调整 prompt 或换模型。
- 如果是内网或离线环境，提前下载好模型和依赖，避免运行时拉取失败。

## 总结

本地 LLM 部署对 OpenClaw 用户来说，解决的是成本、隐私和离线可用性问题。但它不是云端大模型的平替，尤其是小模型在复杂工具调用上仍有明显短板。工程化的做法是：选对量化模型，配置好 OpenAI 兼容接口，严格管理系统提示和上下文，再逐步把 Agent 的日常小任务迁移到本地。能跑起来只是第一步，跑得稳、能接住 MCP 工具链，才是真正可用的状态。

---

