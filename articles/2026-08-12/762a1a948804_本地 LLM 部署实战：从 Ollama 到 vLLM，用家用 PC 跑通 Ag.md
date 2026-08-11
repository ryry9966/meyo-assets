---
title: 本地 LLM 部署实战：从 Ollama 到 vLLM，用家用 PC 跑通 Agent 私有推理
feedId: 32660
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在日常用 Agent 搭建自动化流程时，我们需要反复调用大模型完成推理、总结、工具选择等任务。云端 API 虽然方便，但一旦流程复杂度上升，很快就会碰到几个现实问题：**调用频率受限、网络延迟不稳定、数据隐私要求、以及持续产生费用**。特别是与 MCP 工具链结合后，一个任务可能触发数十次模型调用，API 成本迅速膨胀。

如果有一台配置尚可的家用电脑（哪怕只是 16GB 显存的消费级显卡），完全可以把一部分 LLM 负载迁移到本地，自建推理服务。本文就来记录一套从入门到可接入 OpenClaw/插件系统的实践方法。

## 决定本地部署前的灵魂拷问

在动手前先问自己几个问题，能省下大量调试时间：

- **我的场景真的需要本地模型吗？** 如果只是偶尔用用 Agent，付费 API 的性价比更高。
- **我能接受性能下降吗？** 本地跑 7B/14B 量化模型的推理质量一定弱于 GPT-4，但对结构化输出、分类、简单总结等任务通常够用。
- **我的硬件扛得住吗？** 7B 模型 INT4 量化约需 5–6GB 显存，14B 约需 10–12GB。没有独显就不要强求了，等后续买设备再说。

## 方案选择：两个层级满足不同需求

本地部署主流路线有两条：**Ollama（零门槛）** 和 **vLLM（高性能可定制）**。多数 Agent 框架只需要一个兼容 OpenAI 的 API，两者都能满足。

- **Ollama**：开箱即用，一条命令拉取模型，自动管理量化，CPU/GPU 混合推理，适合快速验证。
- **vLLM**：生产级吞吐量，PagedAttention 显存优化，支持多 GPU 并行，适合高频调用场景。

下面分别演示，重点放在如何与 OpenClaw 等插件系统对接。

## 实战步骤

### 1. Ollama 快速起手

安装：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

拉取模型（推荐 Qwen2.5 系列，中文效果好）：

```bash
ollama pull qwen2.5:7b
```

此时模型已可用，直接测试：

```bash
ollama run qwen2.5:7b
```

关键的一步：暴露 API。Ollama 默认在 `http://localhost:11434` 提供兼容接口，但需要设置环境变量解除局域网限制（按需）：

```bash
export OLLAMA_HOST=0.0.0.0:11434
ollama serve
```

API 调用示例：

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:7b",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### 2. vLLM 高性能部署（适用有 GPU 的机器）

创建 Docker 容器，挂载模型目录，一次性启动服务：

```bash
docker run --gpus all \
  -v /home/user/models:/models \
  -p 8000:8000 \
  vllm/vllm-openai:latest \
  --model /models/Qwen2.5-7B-Instruct-GPTQ-Int4 \
  --dtype float16 \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.95
```

这里选的模型是 GPTQ 量化版本，显存占用仅约 6GB，留出余量给 KV cache。启动后 `http://localhost:8000/v1` 就是一个标准 OpenAI 接口。

### 3. 接入 OpenClaw 或 MCP 工具链

在 OpenClaw 的 LLM 配置中，将 `provider` 设置为 `openai` 兼容模式，填入自定义 endpoint：

```yaml
llm:
  provider: openai
  api_base: http://localhost:8000/v1
  api_key: not-needed
  model: Qwen2.5-7B-Instruct-GPTQ-Int4
```

如果使用 MCP Server 中的 LLM 工具调用，需要确保工具描述的 prompt 与模型对齐。Qwen2.5 的 chat template 对 function calling 有一定支持，建议启用 JSON 模式或使用 guided generation（vLLM 支持 `--guided-decoding-backend` 参数）。

## 踩坑实录

### 坑一：模型选错，功能残疾

想用本地模型做 Agent 的工具调用，直接拉取基础 `qwen2.5:7b` 会缺少指令跟随能力，必须指定 Instruct 版，例如 `qwen2.5:7b-instruct`。Ollama 上模型标签很多，务必看清楚。

### 坑二：Ollama 默认上下文太短

默认 `num_ctx` 为 2048，多轮对话或长文档很快触发截断。启动 serve 时指定：

```bash
ollama serve --num-ctx 4096
```

或在 Modelfile 中固定该参数。

### 坑三：vLLM 启动 OOM，显存越界

设置 `--gpu-memory-utilization` 过高会导致 CUDA out of memory。通常 0.9 是安全上限，尤其开启多卡并行时还需要留出通信缓冲。

### 坑四：OpenAI 兼容但不完全等价

本地部署透出的 API 可能不支持 `seed`、`logprobs`、`stop` 等所有参数。Agent 驱动使用前，最好精简掉非必需参数，或配置 litellm 做一层中间网关进行字段过滤。

## 可复用建议

- **工作负载分层**：复杂推理留给云端大模型，规则匹配、简单分类、文本格式化交给本地 7B/14B 模型，降低整体成本。
- **用 Docker 锁定环境**：避免驱动、CUDA、python 包之间互相打架。Ollama 和 vLLM 都提供官方镜像，直接使用。
- **监控显存与吞吐**：推荐 `nvtop` 或 `gpustat` 实时查看显存占用，`prometheus + grafana` 做长周期统计，防止服务悄悄变慢。
- **为 Agent 准备 fallback**：编写 API 调用逻辑时，加上 try/except，当本地服务不可用时自动切换到云端 API，保障自动化流程不断线。
- **量化选择 Q4_K_M**：在 Ollama 的 Modelfile 或 vLLM 的模型下载中，该量化级平衡了推理速度与效果，7B 模型达到准实时响应。

## 总结

在家用电脑上部署本地大模型不再是玩具，它确实能成为 Agent 和自动化工作流中可靠的一环。用 Ollama 快速验证，再根据吞吐需求决定是否升级到 vLLM，整个过程工程化程度高、可复现。关键要认清能力边界：不对本地模型抱有不切实际的幻想，同时充分利用它“零延迟、零成本、隐私可控”的优势。当你看着监控面板上的推理延迟稳定在 30ms 以下时，会感到这一切折腾都值得。

---

