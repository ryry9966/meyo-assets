---
title: 本地 LLM 部署实践：在家用电脑给 OpenClaw 跑一个可工具调用的后端
feedId: 34255
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

给 OpenClaw/Agent 接本地模型，主要图隐私数据不出本机、降低长任务 token 成本、离线或弱网环境可用。但家用电脑不是推理服务器，显存和带宽都有限，目标不是“跑最大的模型”，而是“用最小资源稳定支持 Agent 的工具调用”。

## 问题

最常遇到的不是模型不能聊，而是一接 MCP/工具就出问题：返回空 `tool_calls`、输出非法 JSON、选错工具或漏掉必填参数。另一个典型问题是显存不足导致上下文极短，或者 OpenClaw 配置 OpenAI-compatible endpoint 后 404、超时、并发跑不动。核心矛盾是：Agent 需要可靠的 function calling 和足够上下文，家用硬件通常只能给到 7B-14B 量化模型。

## 做法/步骤

### 1. 硬件与模型基线

建议 NVIDIA 12GB 显存起步，16GB 更稳；Apple Silicon 建议 32GB 统一内存。模型优先选官方支持 function calling 的 instruct 版，不要用 base 模型。可用基线如下：

- 低配：`Qwen2.5-7B-Instruct` Q4_K_M，约占 4.5-5GB 显存。
- 默认：`Qwen2.5-14B-Instruct` Q4_K_M，约占 9-10GB 显存，建议 16GB 卡跑。
- 量化优先 Q4_K_M；不要直接上 Q2/Q3，工具调用能力会明显下降。

### 2. 推理后端

家用环境用 Ollama 足够，重点是它暴露 OpenAI-compatible `/v1`，并且 GGUF 量化生态全。启动：

```bash
OLLAMA_HOST=127.0.0.1:11434 ollama serve
ollama pull qwen2.5:14b-instruct-q4_K_M
```

用 Modelfile 固定推理参数，避免默认值不合适：

```dockerfile
FROM qwen2.5:14b-instruct-q4_K_M
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
PARAMETER num_batch 512
```

### 3. 先验证工具调用，再接 OpenClaw

不要一上来就在 OpenClaw 里排错。先用 `curl` 验证 `/v1` 和 `tool_calls`：

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
 -d '{"model":"qwen2.5:14b-instruct-q4_K_M","messages":[{"role":"user","content":"What time is it? Use get_time"}],"tools":[{"type":"function","function":{"name":"get_time","description":"Return current time","parameters":{"type":"object","properties":{},"required":[]}}}]}'
```

返回里必须包含 `tool_calls`，且参数合法。如果为空或 JSON 粘连，先别进 OpenClaw。

### 4. 接入 OpenClaw

模型配置按本地 endpoint 填：

- baseURL：`http://127.0.0.1:11434/v1`
- api_key：随便填，例如 `local`
- 模型名：`qwen2.5:14b-instruct-q4_K_M`
- 并发设为 1-2

如果 OpenClaw 跑在 Docker 里，注意 `127.0.0.1` 指向容器自身，宿主机 Ollama 要用 `http://host.docker.internal:11434/v1`。

## 踩坑点

- **baseURL 漏 `/v1`**：这是 404 最常见原因。
- **12GB 卡跑 14B 长上下文 OOM**：必须限制 `num_ctx`，或换 7B。
- **MCP 工具太多导致模型混乱**：一个 MCP server 暴露 20 个工具时，7B/14B 容易选错工具、漏 required 参数。只在客户端暴露任务必需工具，或写过滤层精简 schema。
- **Ollama `/v1` 对 `tool_choice`、`response_format` 等高级参数兼容不完整**：不要依赖强约束，优先用 system prompt 引导。
- **模板敏感**：不要随便套其它模型的 chat template，尤其 Agent 框架改写 system prompt 后效果可能变差。
- **长会话速度断崖**：8K 是家用 14B 的甜点区，超过后 tokens/s 和准确率都会下降。

## 可复用建议

- 先做最小工具调用回归：定义 3 个工具（无参、有必填参、有枚举参），跑 10 次看成功率。7B 常在枚举参数上翻车，14B 成功率明显更高。
- MCP 工具 schema 写短，只保留必要描述。
- 量化用 Q4_K_M 起步，显存允许再考虑 Q5_K_M。
- 显式固定 Ollama 模型 tag，避免 `latest` 升级改变模板或量化文件。
- 本地模型作为隐私/简单任务后端，复杂长任务仍回落远端模型。
- 监控显存：`nvidia-smi -l 1`。出现周期性掉 0 或 OOM，先降 `num_ctx`，不要硬加并发。

## 总结

本地 LLM 部署的关键不是“跑起来”，而是“跑得稳、能调用工具”。从 14B Q4 开始，用 OpenAI-compatible endpoint 接入，先用 `curl` 验证 `tool_calls`，再接 OpenClaw；MCP 工具要精简，并发要小，上下文要限制。做到这几点，家用电脑可以稳定承担中低复杂度 Agent 的隐私任务。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c5ec2a3ed060fba7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/bd842a8f5875164f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/6bc18176145fbdeb.png)

