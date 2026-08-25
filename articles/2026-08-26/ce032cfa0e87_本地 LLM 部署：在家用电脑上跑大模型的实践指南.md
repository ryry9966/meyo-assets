---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 34760
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

对 OpenClaw/Agent/MCP 这类自动化实践者来说，本地 LLM 不是“替代云端大模型”的口号，而更像一个可控的调试后端。开发工具调用、插件链路和 MCP server 时，本地模型能提供离线、可重复、不消耗 API 额度的环境；但家用电脑的显存、内存和推理速度都有限，必须先确认边界再动手。

## 问题

家用电脑跑大模型的真实瓶颈通常不是“能不能跑”，而是：

- 显存/内存不足以加载高精度模型；
- 纯 CPU 推理速度太慢，Agent 多轮调用不可用；
- 小模型的 function calling 不稳定，JSON 输出漂移；
- 本地服务与 OpenAI 兼容接口存在细节差异。

## 做法/步骤

### 1. 硬件自检

先看显存和内存。7B/8B 指令模型的 Q4_K_M 量化版本，权重大约 4-6GB，加上 KV cache 和上下文，建议：

- 8GB 显存：可跑 7B/8B Q4，上下文 8192 左右；
- 16GB 统一内存的 Apple Silicon：可跑 7B/8B，甚至 14B 低量化；
- 无独显、16GB 内存：可跑 3B/4B 或 7B Q4 纯 CPU，但速度只适合 smoke test。

### 2. 选择推理后端

优先 Ollama，跨平台、部署简单、自带 OpenAI 兼容接口，适合 OpenClaw 接入。其次 LM Studio 适合图形化调试，llama.cpp server 适合需要自定义参数的老手，MLX 适合 Apple Silicon。

### 3. 选模型

不要只看榜单，优先选支持工具调用的指令模型。建议从这些开始：

- Qwen2.5-7B-Instruct / Qwen2.5-14B-Instruct
- Llama-3.1-8B-Instruct
- Mistral-7B-Instruct-v0.3

量化优先 Q4_K_M 或 Q5_K_M，避免极限低比特。

### 4. 启动本地服务

以 Ollama 为例：

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```

然后写一个 Modelfile 固化参数：

```dockerfile
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER temperature 0.1
PARAMETER num_ctx 8192
PARAMETER stop "<|im_end|>"
```

创建并运行：

```bash
ollama create local-qwen7b -f ./Modelfile
ollama run local-qwen7b
```

服务默认在 `http://localhost:11434`，OpenAI 兼容端点为 `http://localhost:11434/v1`。

### 5. 接入 OpenClaw/Agent

在 OpenClaw 配置中把模型服务指向本地端点，`api_key` 可以填任意非空值，模型名使用 `local-qwen7b`。先发一个最小 `chat/completions` 请求验证：

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local-qwen7b","messages":[{"role":"user","content":"ping"}]}'
```

再跑一个带 tools 的请求，确认工具调用返回格式是否稳定。尤其对 MCP 工具链路，OpenClaw 会把工具定义转换成 function calling 格式，本地模型必须能输出 `tool_calls`，否则链路会中断。

## 踩坑点

- **上下文不够**：工具定义会占用不少 token，OpenClaw 注入的系统提示、MCP schema 可能超过默认 4096。把 `num_ctx` 设到 8192 或 16384，但要注意 KV cache 显存会随之增加。
- **Function calling 飘 JSON**：小模型容易输出多解释、缺字段或非法 JSON。温度设 0 或 0.1，优先用支持原生 tools 的模型，必要时用 grammar/json mode 约束。
- **CPU 推理太慢**：纯 CPU 跑 7B 时，首 token 可能数秒，Agent 多轮循环会非常痛苦。要么缩小模型，要么减少每轮工具数量，要么把本地模型仅作为开发验证。
- **API 兼容差异**：本地服务可能不支持流式、`tool_calls` 字段返回格式不同。先用 curl 测通，再接入 Agent；必要时用 LiteLLM 做统一代理。
- **停止词与模板**：很多重复输出、吞字问题来自 tokenizer 模板或停止词不匹配。Modelfile 里显式设置 `stop`，不要只依赖模型默认。

## 可复用建议

1. 把模型名、量化版本、温度、上下文长度写进配置文件，避免每次手动调参。
2. 准备两套后端：本地小模型用于快速 smoke test，远程大模型用于正式长时间任务。
3. 对工具调用做最小可复现用例：一个 echo 工具、一个需要参数的 MCP server，通过再扩展。
4. 用 `ollama ps` 或 `nvidia-smi` 监控显存/内存占用，记录 tokens/s 和首 token 延迟，判断瓶颈。
5. 如果只是验证 Agent 逻辑，不一定强上大模型；3B/4B 能暴露流程问题，速度更快。

## 总结

家用电脑跑本地 LLM 的工程价值，在于为 Agent、MCP 和插件开发提供一个离线、可重复的调试环境。它不是万能的云端替代品，但选对量化、固化参数、验证工具调用后，能显著降低开发调试成本。重点不是“跑起来”，而是跑得稳、可复现、能接入现有工具链。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/d7f9fae6ae769710.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/aa5045c53e51d0ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b109aab5185366db.png)

