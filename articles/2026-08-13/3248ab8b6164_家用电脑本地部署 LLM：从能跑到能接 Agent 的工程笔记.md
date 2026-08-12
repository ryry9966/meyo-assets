---
title: 家用电脑本地部署 LLM：从能跑到能接 Agent 的工程笔记
feedId: 32882
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在家用电脑上跑本地大模型，不是为了替代云端旗舰模型，而是给 OpenClaw、Agent、MCP、插件和自动化任务提供一个可离线、可反复压测、可固定版本的调试环境。

常见需求包括：开发 MCP 插件时不想消耗线上 API 额度；自动化流程涉及本地文件或内部数据；需要固定模型版本做回归测试；或者单纯想在断网环境下继续调试工具调用链路。

但家用硬件边界很明确：消费级显卡显存通常在 8GB 到 24GB 之间，内存带宽有限。因此目标不是“跑最大的模型”，而是“跑一个能稳定接住工具调用的最小可用模型”。

## 问题

本地 LLM 部署到家用电脑，真正卡住的往往不是“能不能对话”，而是“能不能接 Agent”。

具体表现为：

- 模型能聊天，但 function calling 格式不稳定；
- 工具定义一多，上下文被系统提示、历史消息、MCP 返回结果迅速撑爆；
- OpenClaw 多步循环产生并发请求，显存峰值翻倍后直接 OOM；
- 本地推理服务与 OpenAI 兼容接口存在细微差异，导致部分参数不生效。

这些问题通常不是模型不行，而是部署和接入方式没有按工程化方式收敛。

## 做法 / 步骤

### 1. 先盘点硬件，再决定模型规模

以消费级 NVIDIA 卡为例：

- 8GB 显存：优先 7B/8B 模型，Q4_K_M 或 Q5_K_M；
- 16GB 显存：可以尝试 14B Q4，或 7B/8B Q8；
- 24GB 显存：可以跑 32B Q4，或 14B Q8。

内存建议不低于 32GB。如果显存不够，可以把部分层卸载到 CPU，但首 token 延迟会明显上升，Agent 场景下体验较差。

### 2. 选择推理引擎

优先使用 Ollama 或 llama.cpp server。

两者都提供 OpenAI 兼容 API，方便 OpenClaw 直接通过 `base_url` 接入。LM Studio 适合手动调参和浏览模型，但做自动化服务时，更推荐命令行启动的常驻服务。

### 3. 选择经过工具调用微调的模型

不要盲目追大模型。优先选明确支持 function calling 的 instruct 模型，例如 Qwen2.5/3 系列、Mistral 系、Llama-3.1 8B Instruct 等。

下载 GGUF 文件时，至少要 Q4_K_M 以上。Q3 及以下在工具调用任务中容易出现参数截断、格式错误或漏参。

启动示例：

```bash
# Ollama
OLLAMA_NUM_PARALLEL=1 ollama serve
ollama run qwen2.5:7b-instruct-q4_K_M

# llama.cpp
llama-server -m model.gguf \
  -c 8192 \
  --n-gpu-layers 99 \
  --parallel 1 \
  --host 0.0.0.0 \
  --port 8080
```

### 4. 接入 OpenClaw / MCP

在 OpenClaw 中配置本地模型端点：

- `base_url`：`http://localhost:11434/v1`
- `api_key`：`ollama`
- 模型名与本地加载名称保持一致

如果 OpenClaw 侧强制使用 `response_format` 或 JSON mode，但本地服务不支持，需要关闭该选项，改为在系统提示中明确要求输出 JSON。

### 5. 做一次最小工具调用回归

用一个极简 MCP server，例如返回当前时间或读取本地文件，连续测试 20 次工具调用。

观察：

- 格式错误率；
- 漏参、错参；
- 幻觉工具名；
- 是否返回多余解释文本。

记录失败样本，后续调 prompt 或更换模型时可以作为对照。

## 踩坑点

- **量化过狠**：Q3 及以下不适合工具调用，优先 Q4_K_M 以上。
- **上下文超长导致 OOM**：Agent 场景中，系统提示、工具定义、历史消息、MCP 返回内容会迅速叠加。显存不足时，优先降低上下文长度，而不是盲目降低量化等级。
- **KV cache 未优化**：可以尝试 `--flash-attn` 或 KV cache 量化，减少显存占用。
- **并发设置过高**：OpenClaw 多步循环可能快速产生多个请求。务必限制 `OLLAMA_NUM_PARALLEL=1` 或 `--parallel 1`。
- **chat template 不匹配**：有些 GGUF 文件自带的模板不对，导致工具调用格式混乱。必要时在 Ollama Modelfile 中手动指定 `TEMPLATE`，或在 llama.cpp 中使用 `--chat-template`。
- **API 兼容性差异**：本地推理服务可能不支持部分 OpenAI 参数。遇到报错时，先确认是服务端不支持，还是 OpenClaw 侧强制传参。

## 可复用建议

1. 固定模型文件、量化版本和启动参数，写进 README 或 compose 文件。
2. 用 `curl` 直接发一条带工具定义的 `/v1/chat/completions` 请求，验证返回 JSON 路径是否符合预期。
3. 本地模型单独设置 `temperature=0` 或 `0.1`，降低随机性对工具调用的影响。
4. 准备云端 fallback：复杂推理仍走云端模型，本地模型只负责结构化提取、离线批处理和工具调用。
5. 记录显存占用、首 token 延迟和工具调用成功率，避免自动任务把服务跑挂后无法复现。

## 总结

家用电脑跑本地大模型，在 2025 年已经足够实用，前提是接受它的边界：模型小、并发低、上下文有限。

对 OpenClaw、Agent、MCP、插件开发来说，本地 LLM 的核心价值不是通用智能，而是低成本复现、离线调试、固定版本回归。先把一个 7B/8B Q4 模型跑稳，再逐步接入工具链，比直接上大模型更容易获得稳定的工程收益。

---

