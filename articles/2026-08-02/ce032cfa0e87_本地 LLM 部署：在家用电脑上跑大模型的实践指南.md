---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 31307
source: 综合讨论
publishedAt: 2026-08-02
---

## 为什么要在本地部署 LLM

在 Agent 和自动化工作流里，API 调用成本、网络延迟和数据隐私正成为硬约束。本地运行大模型能让你的 OpenClaw 或 MCP 工具链在完全离线环境下工作，且无需为每次推理付费——对于高频调用、内部数据处理、敏感文档分析等场景极为实用。

但家用电脑毕竟不是 A100 集群。要让 7B-13B 参数的模型在普通硬件上稳定提供 API 服务，需要处理好量化、显存占用、上下文长度和并发这四个核心问题。本文记录一套我自己踩坑后沉淀下来的部署流程，面向已有 OpenClaw/Agent/MCP/插件开发经验、想把 LLM 后端切换到本地的读者。

## 硬件与模型选型经验

先给一个工程化的“够了”基线：

- **内存**：16GB 可跑 7B Q4 量化，32GB 可跑 13B Q4 量化，建议不低于 32GB。
- **显卡**：NVIDIA 6GB+ 显存可加速 7B，12GB+ 可加速 13B；无独显则纯 CPU 推理（速度慢但可用）。
- **磁盘**：单个 gguf 模型文件 4-15GB，建议 SSD。

**模型选择**：优先考虑对 Agent 任务友好的模型，例如 Mistral-7B-Instruct、Qwen2.5-7B-Instruct、Llama-3-8B-Instruct 等，这些在工具调用、指令遵循上表现稳定。量化版本选 `Q4_K_M`，这是精度-速度-体积的最佳平衡点。Q5、Q6 在 CPU 推理时提升不明显却显著增加内存占用；Q2、Q3 虽快但输出质量下降严重，Agent 用容易产生幻觉和格式错误。

## 部署实践：Ollama + 兼容 API

如果你需要快速接入 OpenClaw 或 MCP 客户端，**Ollama 是目前最省心的方案**——它封装了 llama.cpp，提供模型拉取、参数管理、GPU/CPU 混合推理和 OpenAI 兼容 API。

### 1. 安装与环境准备

Linux/macOS 下一条命令安装：
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
Windows 推荐直接使用原生安装包（已集成 WSL2 后端），避免手动折腾 CUDA 驱动。

启动服务并设置为开机自启：
```bash
systemctl --user enable ollama
systemctl --user start ollama
```

### 2. 拉取模型与量化文件

以 Qwen2.5-7B-Instruct 的 Q4_K_M 量化版为例：
```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```
如果官方没有对应量化标签，可以去 HuggingFace 下载 .gguf 文件，再通过 Modelfile 导入：
```dockerfile
FROM /path/to/model.gguf
PARAMETER temperature 0.5
PARAMETER num_ctx 4096
```
然后 `ollama create mymodel -f Modelfile`。

**踩坑点**：`num_ctx` 默认只有 2048，必须改大，否则 Agent 的多轮对话很快截断。建议设成 4096 或 8192，但每增大一倍，推理时显存/内存用量会显著上升。

### 3. 验证 API 与性能调优

启动模型：
```bash
ollama run mymodel
```
另开终端测试兼容 API：
```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mymodel",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```
确认返回正常后，即可在 OpenClaw 中将 `openai_api_base` 指向 `http://localhost:11434/v1`，`model` 填你自定义的名称。

**性能调优关键参数**：
- **并发**：Ollama 默认单请求独占 GPU。如果需要同时处理多个 Agent 任务，记得在启动时设置 `OLLAMA_NUM_PARALLEL=4`（环境变量）并增加 `num_predict` 上限。
- **显存控制**：纯 CPU 推理时不必操心。如果有混合推理，可通过 `OLLAMA_GPU_LAYERS` 限制加载到 GPU 的层数，防止 OOM。例如 `OLLAMA_GPU_LAYERS=24 ollama serve`。

### 4. 与 MCP 及自动化流程对接

如果你在用 MCP server 为 Agent 提供工具，本地模型同样可以作为“胶水”：
- 将复杂的对话生成交给稳定的大模型（哪怕在云端），本地 7B 模型负责**高频率、低延迟、低风险的意图分类、实体提取、敏感数据脱敏**。
- 通过 OpenAI 兼容接口，直接在你的 MCP 工具函数里用 `openai` SDK 调用本地服务，无需额外适配层。

替换原有云端 API 的唯一改动就是将 `base_url` 切到本地。这种无缝切换对于需要频繁回归测试、离线环境或内网部署的自动化流水线非常有价值。

## 核心踩坑与突破

1. **显存溢出 (OOM)**  
   即使量化后，半精度模型在某些显卡上仍可能爆显存。解决思路：减少 `num_ctx`，降低 `OLLAMA_GPU_LAYERS`，或换用量化更激进的 `Q4_0`（牺牲部分质量）。

2. **CPU 推理慢得像打字机**  
   7B Q4 模型在普通 8 核 CPU 上大约 5-8 token/s，勉强可用但交互感差。开多并行会导致 token/s 雪崩式下降。建议 CPU 只做任务队列中非实时的批量推理（如夜间日志分析）。

3. **上下文长度陷阱**  
   模型本身宣称支持 32K，但物理内存没分配这么多，实际跑到 6K 就报错。务必通过 `num_ctx` 精确限制，并按 `num_ctx * 量化位宽 / 8` 估算额外内存，确保不超过物理内存 + 显存上限。

4. **模型缓存与磁盘空间**  
   Ollama 下载的模型和生成的层缓存会占大量空间，记得定期清理不再需要的模型（`ollama rm`），否则磁盘悄悄爆满。

## 可复用建议

- **先云后本**：开发阶段用云端昂贵模型验证逻辑，正式部署切换到本地 7B/13B 作为降级或常态化后端。
- **建立一个模型 profile 库**：记录每个模型在你机器上的 `num_ctx`、GPU layers、平均 token/s、内存占用，方便快速匹配任务。
- **监控与熔断**：给你的 Agent 加一个推理计时与错误计数逻辑，当本地服务出现超时或返回格式错误超过阈值时，自动回退到云端 API，保证自动化不中断。
- **使用 llama.cpp server 做极端定制**：当 Ollama 的默认策略无法满足并发或资源调度需求时，可以退回到直接使用 `llama-server`，通过参数 `-c 4096 -ngl 24 --parallel 4` 精细控制。

## 总结

在家用电脑上部署 LLM 远非“下载-运行”这么简单，而是一次硬件理解、模型量化、服务化封装和 Agent 工程调度的复合实践。一旦跑通，你就拥有了一个完全自主可控、零边际成本的推理后端，尤其适合**嵌入式 Agent、离线自动化、本地敏感数据处理**等场景。

对于 OpenClaw 社区的开发者而言，本地 LLM 不仅是一个省钱选项，更是将自动化推向离线、实时、隐私合规的更佳手段。建议从 Ollama 起步，先在一个 7B 模型上走通整条 API 链，再根据实际负载决定是否升级硬件或切换量化策略。

---

