---
title: 在家用电脑上跑大模型：面向 OpenClaw / Agent 用户的本地 LLM 部署实战
feedId: 31388
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：为什么在本地跑 LLM
如果你正在用 OpenClaw 搭建 Agent，大概率已经习惯了通过 API 调用 GPT-4 这类公有云模型。但三个现实问题会推着你考虑本地部署：

1. **隐私与合规**：某些 MCP 工具会传递代码、文档片段或内部数据，发送到云端不符合安全策略。
2. **延迟与配额**：Agent 的多轮工具调用会产生大量串行请求，云 API 的速率限制和网络波动会让任务跑一半就卡住。
3. **开发与调试成本**：频繁调整 prompt、测试新的 function calling 模式时，本地近乎零成本的推理能省下可观的 API 开销。

这篇指南的目标读者是有 OpenClaw / Agent 开发经验，想在本地搭建可控推理环境的工程师。我们不讨论“本地模型能不能替代 GPT-4”这种泛话题，而是直接聚焦一套可复现、能用起来的方案，并重点说明在 Agent 场景下你大概率会踩的坑。

## 要解决的问题
本地部署 LLM 不只是把模型下载下来随便跑跑。在 Agent 上下文里，你需要一个服务能够：

- 提供 OpenAI 兼容的接口，以便直接接入 OpenClaw 的模型配置。
- 支持 **工具调用（function calling）**，这是 Agent 与 MCP 工具交互的核心能力。
- 在消费级硬件（单张 24GB 显存的游戏卡，甚至纯 CPU 的 MacBook）上以可接受的推理速度工作。
- 保证一定长度的上下文窗口，以免 Agent 的多步推理在中间就因为上下文截断而崩溃。

## 方案选型与搭建步骤
目前社区最成熟的本地推理方案是 **Ollama + 支持 function calling 的开源模型**。结合 Open WebUI 或 LiteLLM 做代理，可以无缝接入 OpenClaw。

### 第一步：安装 Ollama 并拉取模型
```bash
# 各平台安装参考 https://ollama.com
ollama serve   # 启动服务
ollama pull llama3.1:8b-instruct-q4_K_M  # 示例：8B 参数，4-bit 量化
```
这里直接选了 `instruct` 标签的版本，因为它内置了 prompt template，能较好遵循 system prompt 与工具调用格式。8B 模型在 16GB 统一内存的 Mac 上就能跑，纯 CPU 推理也有约 10–15 token/s，够做开发调试。

### 第二步：选择支持工具调用的模型
官方 Llama 3.1 8B 虽然支持 function calling，但在实际 Agent 场景中遵循度不稳定。一个更务实的路线是使用 **Mistral 7B v0.3** 或专为工具调用微调的 **NousResearch/Hermes-2-Pro**。在 Ollama 中可以直接拉取社区维护的 Modelfile：
```bash
ollama pull hermes2pro:7b-llama3-q4_K_M
```
这类模型在给定 OpenClaw 的标准工具调用格式（通常是 JSON function description）后，能稳定输出符合格式的 `tool_calls` 结构。

### 第三步：暴露 OpenAI 兼容接口并接入 OpenClaw
Ollama 默认端口 11434，但直接用它接入 Agent 框架有时会因路径不匹配出错。推荐在中间加一层 **LiteLLM proxy**：
```python
# litellm_config.yaml
model_list:
  - model_name: local-agent
    litellm_params:
      model: ollama/hermes2pro:7b-llama3-q4_K_M
      api_base: http://localhost:11434
      temperature: 0.1
```
启动代理：`litellm --config litellm_config.yaml --port 4000`。然后在 OpenClaw 中将模型地址设为 `http://localhost:4000`，API Key 留空。这样你就获得了一个与 OpenAI /v1/chat/completions 完全同构的本地端点。

### 第四步：在 OpenClaw Agent 中验证
用内置的 `tool_use` 测试用例跑一遍：让 Agent 调用一个模拟的 MCP 工具，观察模型能否正确解析工具描述、填充参数并在收到工具返回值后继续推理。这一步能暴露大部分不兼容问题，不要等到真实任务才调试。

## 踩坑记录与复盘
在实际接入 OpenClaw 的过程中，以下几个坑点几乎是必现的：

**坑 1：工具调用格式解析失败**  
开源模型对 function calling 的输出方式五花八门。有的输出纯文本 JSON，有的漏掉 `function` 关键字。解决方法：在 Ollama Modelfile 中明确设置 `template` 为对应的 chat template（如 `llama3.1` 或 `mistral`），同时将 `return_full_text` 参数设为 false，要求只返回生成部分。

**坑 2：上下文窗口快速耗尽**  
Agent 每轮对话都会把工具定义、历史调用结果塞进上下文。8B 模型默认 8k 窗口，一旦工具返回大段 JSON，几轮后就会触发截断。应对策略：在 Agent 配置中启用**上下文压缩**（OpenClaw 支持 summary buffer），或者在 MCP 工具侧对返回数据做裁剪，只保留关键字段。

**坑 3：GPU 内存不足导致切换回退**  
如果你在 12GB 显存的卡上硬跑 7B 模型（int8 量化需约 7–8GB），Agent 多轮对话时 KV cache 可能把剩余空间吃满，导致进程被 kill。务必用 `ollama ps` 监控内存，并选择合适的量化级别：24GB 卡用 `q5_K_M`，12GB 卡用 `q4_K_M`，无独显的机器直接跑 `Q2_K` 或调用纯 CPU 推理。

**坑 4：Response 延迟导致 Agent 超时**  
本地模型首 token 延迟可能达到 2–5 秒（取决于 prompt 长度），而 OpenClaw 默认的 HTTP 请求超时可能设得比较短。如果日志中出现大量 ReadTimeout，调整 Agent 的 `timeout_seconds` 到 120 或更大，并给 LiteLLM 加上重试逻辑。

## 可复用的工程建议
1. **固化工具定义**：不要每次请求都动态生成 tools 数组。将它们做成一个固定的 JSON Schema 文件，在模型启动时一同加载到 system prompt 中，减少推理开销并提升一致性。
2. **采用多模型分层策略**：用本地模型做轻量解析、意图判断的快速路径，复杂推理再交给云端模型。OpenClaw 支持路由规则，可以按任务类型指向不同模型。
3. **监控与回退**：在 LiteLLM 前面加一个健康检查脚本，如果本地服务挂掉就自动切换到备用 API。这和 Agent 里的 fallback 机制配合起来，可以做到对用户无感知。
4. **容器化整个推理栈**：把 Ollama、LiteLLM 和模型文件打包成 Docker Compose 模板，新人加入时只需修改环境变量里的模型路径。这比每个人从零配置 Modelfile 高效得多。
5. **不要轻视量化**：在本地场景，量化不只是为了省钱而是为了“能跑”。Q4_K_M 量化对推理质量的影响远小于上下文截断导致的 Agent 失败，优先保证窗口长度比保持参数精度更重要。

## 总结
本地部署 LLM 并接入 OpenClaw Agent 系统，在 2025 年的工具链支持下已经是一个 80 分位的可行方案。核心路径是：Ollama 提供推理运行时 → 专门微调的工具调用模型提供能力 → LiteLLM 提供标准接口 → OpenClaw 无感接入。真正的挑战不在搭建，而在后续的稳定性调校：控制上下文膨胀、保证工具调用格式一致、合理分配硬件资源。

把这一套跑通之后，你会获得一个完全离线、零边际成本、响应可预测的推理节点。它不替代云端模型，但可以让你的 Agent 开发变得非常实在——随时修改、随时测试，不用心疼 token 消耗。对于做自动化实践的 OpenClaw 用户来说，这是一件值得放进工具箱的利器。

---

