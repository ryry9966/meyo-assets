---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 31121
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：为什么要在本地跑大模型？

给 OpenClaw 接上大语言模型，通常首选云端 API，但在以下场景中，本地部署是更务实的选项：

- **数据敏感**：Prompt 中包含的密钥、内部文档或客户数据，不希望流出本地网络。
- **离线/低延迟要求**：Agent 在工控机或家庭实验室中运行，需要亚秒级首 token 延迟，且不受外网波动影响。
- **成本控制**：高频调试 MCP 插件或长自动化流程时，云端费用会线性增长，本地一次硬件投入后近乎零边际成本。
- **可定制性**：需要微调专属模型（如内部知识库、运维脚本补全），私有化部署是基础。

这篇指南会**聚焦与 OpenClaw 生态的衔接**，不讨论训练，只讲在家用电脑上完成一个可供 Agent 稳定调用的本地 LLM 服务。

## 问题拆解：主要挑战是什么？

在家用电脑上跑大模型，瓶颈在于：

- 消费级 GPU 显存有限（多数 8–24GB），好一点的 CPU+RAM 组合也可行，但吞吐量会大幅下降。
- 开源模型尺寸与精度需要仔细选择，7B-13B 是甜区。
- Agent 场景要求 **长上下文**（至少 8k token）和 **结构化输出**（工具调用），这对量化模型提出了额外要求。
- OpenClaw 通过 OpenAI 兼容 API 接入 LLM，因此需要本地推理引擎提供相同的 HTTP 接口，并处理流式输出。

## 实践步骤

### 1. 选择模型与量化等级

重点考虑这些模型（HuggingFace 直接可获取）：

- **Mistral 7B** / **Mixtral 8x7B**（MoE，8x7B 性能好但资源要求高）
- **Meta Llama 3 / 3.1** (8B)
- **Qwen 2.5** (7B/14B) —— 中文和多语言支持优秀，Agent 场景稳定
- **通义千问 Qwen 2.5-Coder** (7B) 对代码和工具调用更友好

量化推荐使用 **GGUF 格式**，等级 `Q4_K_M` 是最佳性价比：在 8B 模型上仅需约 5–6 GB 显存/内存，且质量损失可接受。下载时直接从 TheBloke 等量化仓库拉取 `.gguf` 文件。

### 2. 部署推理服务（Ollama 最省心）

方案一：**Ollama**（推荐）  
```bash
# 假设已下载 qwen2.5:7b 的 GGUF，记入 Modelfile
ollama create my-qwen -f Modelfile
ollama serve
```
Ollama 默认在 `http://localhost:11434` 提供 OpenAI 兼容 API，端点 `POST /v1/chat/completions` 可直接作为 OpenClaw 的 Custom LLM 接入。

方案二：**llama.cpp server**（需要更细粒度控制时使用）  
```bash
./llama-server -m models/qwen2.5-7b-Q4_K_M.gguf -c 8192 --host 0.0.0.0 --port 8080
```
llama.cpp 支持更激进的上下文长度和 KV 缓存量化，适合压榨硬件极限。

### 3. 接入 OpenClaw

在 OpenClaw 的配置中新建一个 LLM Profile，选择 `Custom OpenAI-compatible` 类型：

- **Base URL**: `http://localhost:11434/v1`（Ollama）或 `http://localhost:8080/v1`
- **API Key**: 可任意填写（Ollama 默认不校验，但字段必须非空）
- **Model Name**: `my-qwen`（与 Ollama 创建的名称匹配）

测试时发送一个简单的 “Hello” 消息，确认流式输出和 token 计数正常。

### 4. 与 Agent / MCP 结合的关键点

Agent 会通过 OpenClaw 向本地模型发送系统提示、工具定义和上下文历史。本地模型需支持足够的 token 窗口和遵循 `json_object` 或 `tool_call` 格式。实测中：

- Qwen 2.5 系列对 `tools` 调用支持良好，可稳定返回结构化 JSON。
- 若使用 Llama 3 8B，需要显式在系统提示中注入“你必须使用以下函数……”的指令，并适当降低温度（0.3）以保证格式规整。

## 踩坑点

- **显存不足**：即使是 Q4_K_M，8B 模型也需要约 6GB 显存。如果同时加载 MCP 的本地服务（如 ChromaDB），显存峰值容易超限。建议将模型放在 CPU 上（只用 RAM），牺牲 30% 速度换取稳定性。在 Ollama 中通过 `num_gpu_layers` 控制。
- **API 延迟爆炸**：当上下文接近 8k token，纯 CPU 推理的单个请求耗时可能超过 30 秒。Agent 的超时设置必须放宽，或者限制 `max_tokens` 在 1024 以内。
- **模型不会停止输出**：部分 GGUF 量化版本在处理特殊标记时不佳，导致对话未在 `<|im_end|>` 处停止。可以在 Ollama 模版文件中明确设置 `stop` 序列。
- **OpenClaw 的身份确认失败**：一些本地模型没有很好的 “assistant” 角色训练。遇到连续重试时，需在 Modelfile 中显式设置 `TEMPLATE """{{ if .System }}<|im_start|>system 
{{ .System }}<|im_end|>{{ end }}..."""`，并与模型原生格式对齐。

## 可复用建议

1. **固化一套“本地 Agent 配置包”**：为不同场景准备 Modelfile、system prompt 模板和 OpenClaw profile 的 YAML 导出。下次换电脑只需启动服务并导入配置。
2. **用 Docker 打包推理服务**：将 Ollama 或 llama.cpp 及其模型文件打包成镜像，绑定端口和模型目录。这样可随时在 NAS、迷你主机上迁移。
3. **监控脚本**：写一个简单的健康检查脚本，定期调用 `/v1/models` 并测试完成一个最小闭环（发送 prompt 并获得指定格式内容），失败时重启服务。对于长时间运行的 Automation Agent 特别重要。
4. **分级使用**：高频、低延迟的实时交互仍用云端 API；批量处理、离线文档抽取、预填知识库等长任务切换到本地模型，兼顾效率与成本。

## 总结

在家用电脑上部署大模型并连接 OpenClaw 已经高度工程化，但需取舍：**精度/速度 vs 隐私/成本**。7B-8B 量化模型 + Ollama 是最平滑的起点，Agent 用户应额外关注工具调用合规性和长上下文稳定性。记住：本地 LLM 不是云端竞品，而是你自动化工作流里的可控、低摩擦的专用引擎。把硬件、模型、配置做成标准化的“模块”，就能在需要时随时嵌入到 OpenClaw 的插件链中。

---

