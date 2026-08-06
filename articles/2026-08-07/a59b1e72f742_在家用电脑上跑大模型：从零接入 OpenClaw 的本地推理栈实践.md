---
title: 在家用电脑上跑大模型：从零接入 OpenClaw 的本地推理栈实践
feedId: 31912
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

运行本地大模型，在 Agent 和自动化场景里价值很明确：**零网络延迟、数据不外泄、完全离线可用，还可以无限调用**。当 OpenClaw 需要频繁驱动意图分类、工具选择、长上下文总结时，把推理放在本地也就成了自然选择。

但家用环境通常只有一张消费级显卡（甚至只有 CPU），内存/显存都吃紧，而模型却是越做越大。本文目标是在这种受限资源下，搭建一套能稳定服务 OpenClaw Agent / MCP 插件 / 自动化管线的本地推理栈。

## 问题拆解

单机跑 LLM 主要会撞上四堵墙：

1. **显存墙**：7B 模型 FP16 就要约 14 GB 显存，20B 以上直接超出 RTX 4090 容量上限。
2. **吞吐墙**：纯 CPU 推理速度慢，一个 7B 模型可能只有 3–5 token/s，无法支撑实时 Agent。
3. **上下文墙**：长上下文推理时内存占用和计算量暴增，家用显卡容易直接 OOM。
4. **工程兼容性**：不同框架导出的 API 不完全一致，接入 OpenClaw 时容易出现字段错配（如 `finish_reason`、`tool_calls` 格式）。

下面会沿着 **选框架 → 选模型 → 部署 API → 接入 OpenClaw** 的主线，补齐可落地的工程细节。

## 步骤

### 1. 评估可用资源

先在真实硬件上测一下空闲显存和内存，不要依赖 `nvidia-smi` 上报的总容量，因为桌面环境、浏览器都会占掉一块。

- **有 NVIDIA 显卡**（≥6 GB 显存）：优先用 GPU 推理，量化选 4-bit（GPTQ/AWQ/GGUF q4_K_M）。
- **M 系列 Mac（≥16 GB 统一内存）**：用 Metal 加速，CPU+GPU 混合推理。
- **纯 CPU（≥32 GB 内存）**：只能用 llama.cpp 跑 4-bit 量化，接受偏低的速度。

当前性价比最高的工程基线是：**一张 12 GB 显存的显卡 + 32 GB 系统内存**。这个配置能稳定跑通 7B–13B 量化的模型，并且余量够开较长的上下文窗口（4096–8192 tokens）。

### 2. 推理框架选择

家庭部署首选 **Ollama**，其次是 **llama.cpp** 直调。不推荐直接用 vLLM 或 TGI，因为它们更偏向高并发服务端，对单卡家用环境启动成本高、显存占用也更激进。

- **Ollama**：自带量化模型仓库，一条 `ollama pull` 就能下载，默认提供标准化 OpenAI 兼容 API，省心程度最高。
- **llama.cpp**：更底层，适合需要自己切分 layers 到多 GPU、精确控制 GPU offload 的场景。当需要跑 70B 模型并且用两张卡切分时，用它的 `-ngl` 参数会灵活很多。

本文主线默认使用 Ollama，因为与 OpenClaw 集成最顺滑。

### 3. 模型选择与下载

家用电脑不要追最新、最大的模型，而要追**指令跟随能力强、尺寸合适**的。以下选择经实测在工具调用和结构化输出上表现稳定：

- **通用对话/意图**：`llama3:8b-instruct-q4_K_M`（4-bit 量化，约 5.8 GB）
- **工具调用特化**：`mistral:7b-instruct-v0.3-q4_K_M` 或 `codellama:7b-instruct`
- **中文场景**：`qwen2:7b-instruct-q4_K_M`

下载命令：

```bash
ollama pull llama3:8b-instruct-q4_K_M
```

完成后用 `ollama list` 确认本地已有模型。

### 4. 启动 API 服务并配置稳定参数

直接运行 `ollama serve` 启动 API，但务必通过环境变量控制上下文大小，避免 OOM：

```bash
OLLAMA_NUM_PARALLEL=1 \
OLLAMA_MAX_LOADED_MODELS=1 \
OLLAMA_CONTEXT_LENGTH=4096 \
ollama serve
```

- `OLLAMA_NUM_PARALLEL=1`：强制串行请求，避免家用卡并发两三个请求就爆显存。
- `OLLAMA_MAX_LOADED_MODELS=1`：同时只保留一个模型在显存中。
- `OLLAMA_CONTEXT_LENGTH=4096`：按需设置，Agent 场景通常 4096 足够；如果要长文档可上调，但必须先验证不会 OOM。

如果需要远程访问（比如 OpenClaw 在其他容器里），让 Ollama 监听 `0.0.0.0`：

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

### 5. 接入 OpenClaw

OpenClaw 配置大模型的后端时，直接用 Ollama 提供的 OpenAI 兼容端：

- API base URL：`http://<宿主机IP>:11434/v1`
- API Key：随便填一个字符串（Ollama 不校验）
- 模型名：写 Ollama 中的模型名，例如 `llama3:8b-instruct-q4_K_M`

在 OpenClaw 配置文件（如 `agent.yaml` 或环境变量）中设置：

```yaml
llm:
  provider: openai
  model: llama3:8b-instruct-q4_K_M
  api_base: http://192.168.1.100:11434/v1
  api_key: ollama
  temperature: 0.1
```

注意：部分工具的 `tool_calls` 字段在 Ollama 返回中可能小写为 `tool_calls`，而 OpenClaw 期望 `ToolCalls`。如果出现工具调用解析失败，在 OpenClaw 侧启用 `compatibility_mode: ollama`（具体选项参考 OpenClaw 文档），或者用 litellm 做一层翻译代理。

### 6. 可选：用 litellm 统一代理

如果同时需要本地模型和部分云端模型（如 GPT-4 做复杂规划），可以用 litellm 包装一层统一接口：

```bash
docker run -d -p 4000:4000 ghcr.io/berriai/litellm:main-latest \
  --model ollama/llama3:8b-instruct-q4_K_M \
  --api_base http://宿主机IP:11434
```

OpenClaw 只需指向 litellm 的 `http://localhost:4000`，之后的模型切换、fallback、日志监控都放在 litellm 上完成。

## 踩坑记录

1. **Ollama 启动后第一次请求奇慢**：模型需要加载到显存，这个冷启动延迟通常 20–40 秒。解决方案是准备一个预热脚本（定期调用或配置 `OLLAMA_KEEP_ALIVE` 环境变量延长驻留时间）。
2. **上下文溢出直接 OOM**：即使设置了 `OLLAMA_CONTEXT_LENGTH`，某些模型内部可能因为 KV cache 碎片化而超出预算。一个实用的防御手段是监控 GPU 显存，一旦接近上限（例如 95%），在 Agent 里主动裁剪历史消息。
3. **连续工具调用丢失信息**：因为本地模型指令跟随弱于 GPT-4，在连续 5–6 轮工具调用后容易出现“忘记”前面返回的数据。建议在 Agent 端设定最大连续工具调用数 ≤3，回落到模型重新总结后再继续。
4. **API 返回格式与 OpenAI 不完全一致**：特别是流式模式，OpenClaw 客户端可能预期 `delta` 字段为 `content`，而 Ollama 返回的是 `message.content`。排查这类问题最快的方式是用 `curl` 直接看原始 JSON，然后对齐 OpenClaw 期望的 schema。

## 可复用建议

- **把部署变成 Docker Compose 单文件**：包含 Ollama 和 litellm，用 `init` 脚本预热模型，复制到任何一台家用机器即可上线。
- **显存预算先留 1–2 GB buffer**：不要用到理论最大值，给系统 cuda context 和临时计算留空间。能用 8 GB 显卡跑 4-bit 的 7B 就不要试图塞 13B。
- **接入 OpenClaw 前先做火线测试**：写一个简单的 Node.js/Python 脚本，用这个模型完成 3 轮工具调用，验证 JSON 结构是否正常。这个步骤能提前筛掉 90% 的问题，不要把 Debug 留到 Agent 跑起来之后。
- **本地模型 + 远端 fallback**：用 litellm 配置路由策略，当本地推理超时或报错时自动降级到远端小模型 API，避免自动化管线中断。

## 总结

家用电脑跑大模型服务于 OpenClaw，不是一个“装上就能跑”的玩具任务，而是一个需要仔细分配显存、控制上下文、校验输出格式的小型工程。按本文路径走下来，可以得到一个 **延迟可预测、数据不外流、无限调用零成本** 的本地推理节点。把它作为 Agent 的常态化后端，再加一条远端 API 做备份，就足够支撑大部分室内自动化与插件的日常运行。

---

