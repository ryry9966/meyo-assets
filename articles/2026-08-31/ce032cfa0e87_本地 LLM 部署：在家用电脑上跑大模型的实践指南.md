---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 35543
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景：为什么要在本地跑模型

对 OpenClaw、Agent、MCP 与插件自动化这类实践来说，模型服务通常是云端 API。但云端 API 有成本、延迟、隐私、离线场景等限制。家用电脑本地部署 LLM，可以作为 Agent 的私有推理后端，也可用于敏感数据测试、插件调试、以及在没有公网的环境下快速原型。

但家用电脑不是数据中心，显存和内存是硬约束。这篇记录我在一台中端家用机器上部署本地 LLM 并接入 Agent 工具链的实践过程，重点是可复用的工程化选择，而不是跑分。

## 问题：家用环境的约束

最常见的配置大致两类：N 卡 8-12GB 显存（如 3060/4060/4070），或使用 CPU 推理的核显机器。直接加载 13B/14B 全精度模型基本会 OOM，7B/8B 也需要量化。同时，本地服务需要提供 OpenAI 兼容 API，方便 OpenClaw、MCP 客户端、插件脚本统一调用。另一个容易被忽略的问题是上下文长度：默认值往往偏小，影响 Agent 的多轮工具调用。

## 做法与步骤

**1. 选型：从 GGUF 量化模型入手**

个人推荐先用 llama.cpp 生态，不推荐一开始就上 vLLM。vLLM 对 CUDA 版本和算子编译要求高，适合有稳定显卡和批处理需求；llama.cpp 或 Ollama 对硬件更包容，也支持 CPU/GPU 混合推理。模型优先选 7B/8B 的 instruct 版本，量化选 Q4_K_M 或 Q5_K_M。例如 qwen2.5:7b-instruct-q4_K_M 或 llama3.1:8b-instruct-q4_K_M。

**2. 启动本地 OpenAI 兼容服务**

如果使用 Ollama，拉取后通过环境变量设置上下文，再启动服务：

```bash
OLLAMA_CONTEXT_LENGTH=8192 ollama serve
# 拉取模型
ollama pull qwen2.5:7b-instruct-q4_K_M
# 运行
ollama run qwen2.5:7b-instruct-q4_K_M
```

Ollama 默认会监听 `http://localhost:11434`，OpenAI 兼容端点为 `/v1`。

如果使用 llama.cpp server，命令大致如下：

```bash
./llama-server -m models/qwen2.5-7b-instruct-q4_k_m.gguf \
  -ngl 999 -c 8192 --host 0.0.0.0 --port 8080
```

`-ngl 999` 表示尽可能把层加载到 GPU。显存不够时，可以减小到 30-40，让部分层回退到 CPU。

**3. 接入 OpenClaw / Agent / MCP**

在 OpenClaw 的模型配置中，把 `base_url` 指向本地服务，如 `http://127.0.0.1:11434/v1` 或 `http://127.0.0.1:8080/v1`。API key 可以填任意非空字符串，本地服务通常不校验。优先测试 `/v1/chat/completions` 和 `/v1/models` 两个端点，确认响应结构和字段兼容。MCP 客户端或插件脚本也可以通过环境变量 `OPENAI_BASE_URL` 复用同一本地端点。

**4. 验证与调参**

先用 curl 或 Python requests 做一次最小调用，确认返回的 `choices[0].message.content` 正常。调整参数时注意：本地模型对温度敏感，Agent 场景建议温度 0.1-0.3，避免工具调用格式漂移。上下文长度与显存成正比，8GB 显存 7B Q4 模型通常可以稳定到 4096-8192 token，再高可能触发 KV cache OOM。

## 踩坑点

- **Ollama 默认上下文只有 2048**：很多 Agent 多轮对话或工具输出会超过这个值，导致截断或格式错误。务必显式设置 `OLLAMA_CONTEXT_LENGTH` 或在 Modelfile 中设置 `num_ctx`。
- **显存不足时不要盲目调大 `-ngl`**：llama.cpp 把所有层都加载到 GPU 后，推理时可能因为剩余显存不够分配 KV cache 而直接 OOM。合理做法是先全 GPU 加载，如果 OOM 再逐步降低 `-ngl` 或减小 `-c`。
- **模型输出格式不符合工具调用**：本地小模型对 function calling / JSON schema 支持不如大模型稳定。建议在 system prompt 中明确输出格式，并在 Agent 侧做一层解析容错。不要依赖模型 100% 返回合法 JSON。
- **CPU 推理速度慢**：如果没有独显，纯 CPU 跑 7B Q4，每秒可能只有几 token。用作 Agent 后端只能做轻量工具和短回复，不要期待实时交互。
- **端口和防火墙**：本地服务默认绑定 `127.0.0.1`，如果需要给局域网内其他机器或容器访问，要显式 `--host 0.0.0.0` 并注意安全。

## 可复用建议

1. **固定一个本地端点**：所有工具、插件、Agent 统一走 `localhost:11434/v1` 或 `localhost:8080/v1`，切换模型只需改服务端，不用改每个客户端配置。
2. **从 7B Q4 开始，不要一上来追求大模型**：先跑通链路，再考虑换 13B/14B 或提高量化精度。
3. **记录模型配置**：模型名、量化类型、上下文长度、温度、`-ngl` 等写入配置文件或注释，避免过几天忘记参数。
4. **为 Agent 单独准备一个轻量模型**：如果机器还要同时跑其他任务，可以再启动一个 1.5B/3B 的模型作为指令路由或分类器，主模型只处理复杂推理。
5. **监控显存和内存**：用 `nvidia-smi` 或 `htop` 观察加载和推理时的占用，快速判断是否需要调整分层加载。

## 总结

家用电脑本地部署 LLM 不是替代云端 API，而是提供一种可控、离线、低成本的私有推理端点。对于 OpenClaw、Agent、MCP 和插件自动化实践，它的价值在于让整套工具链在没有外部依赖时依然能跑通、能调试。务实路线是：7B/8B 量化模型 + llama.cpp/Ollama + OpenAI 兼容 API + 显式上下文长度，先跑通，再按需调整。不要被“在家跑大模型”的噱头带偏，工程化的关键是稳定的端点、明确的资源边界和可复用的配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c951cc86fccee9f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/7dcb4ae10c7c97c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/37d3e08c0d9c60b0.png)

