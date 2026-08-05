---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 31686
source: 综合讨论
publishedAt: 2026-08-05
---

# 本地 LLM 部署：在家用电脑上跑大模型的实践指南

## 背景

OpenClaw-CN 社区里 agent 工作流越来越复杂，但多数人仍把 LLM 调用挂在云端 API。对隐私敏感的自动化任务（本地文档摘要、敏感数据提取），或者想控制长期运行成本，本地部署几乎是必经之路。这篇文章记录我在家用电脑部署本地模型并接入 agent 工作流的完整实践，配置可复现。

## 核心问题

家用电脑跑大模型，先要打破一个误解：不是"显存不够就完蛋"，而是"先想清楚跑什么模型、多大上下文、要不要工具调用能力"。

我的基准配置：

- CPU: AMD R7 5800X
- 内存: 32GB DDR4
- GPU: RTX 3070 8GB VRAM
- 系统: Ubuntu 22.04

中等偏下的配置，足以支撑日常 agent 自动化。

## 做法与步骤

### 1. 推理引擎选型

主力用 Ollama，理由：`ollama run qwen2.5:7b-instruct-q4_K_M` 声明式拉取模型，配合 OpenAI 兼容接口切换 agent 后端非常方便。Llama.cpp 作为备选，用于需要精细控制量化参数或做性能调试的场景。

安装：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 2. 模型选择矩阵

多轮对比后的结论：

- **8B 级 Q4_K_M 量化**（Llama 3.1 8B、Qwen 2.5 7B）：日常首选，显存约 6GB
- **14B 级 Q4 量化**：8GB 显存跑长上下文会溢出，需用 `OLLAMA_GPU_LAYERS` 分流到 CPU
- **工具调用**：优先 Qwen 2.5 系列，tool call 稳定性实测高于同量级 Llama

### 3. 接入 Agent 工作流

以 OpenClaw 风格的 agent 配置为例，关键是通过 OpenAI 兼容端点暴露本地模型：

```yaml
llm:
  provider: openai-compatible
  base_url: http://localhost:11434/v1
  model: qwen2.5:7b-instruct-q4_K_M
  api_key: ollama  # 本地无需真实 key
  temperature: 0.2
```

MCP 服务器同理，把 Tool 调用转发到本地模型。实测单 agent 会话（10 轮内工具调用链）延迟 3-8 秒，可接受。

## 踩坑记录

1. **量化不等于无损**：`q2_K` 以下模型中文任务退化严重，代码生成没法看。至少 `q4_K_M` 起步。
2. **上下文是隐性成本**：Ollama 默认 `num_ctx` 只有 2048，长文档任务必须显式调大，否则 agent 三四轮工具调用后"失忆"。
3. **并发是大坑**：Ollama 默认并发为 1，多 agent 并行会排队。设 `OLLAMA_NUM_PARALLEL=2` 可缓解，但 8GB 显存下被卸载到 CPU 的层会让速度骤降。
4. **工具调用不稳定**：量化模型不一定带 function calling 能力。用 `ollama show` 确认 capability 中包含 `tools`，否则 MCP 工具链会静默失败。

## 可复用建议

- 写一个 `start-local-llm.sh`，固定 `OLLAMA_HOST=0.0.0.0 OLLAMA_NUM_PARALLEL=1`，方便局域网内多机共用 agent 后端
- 用 `ollama ps` 监控显存；swap 飙升说明该换更小模型
- 配置中保留 `LOCAL_LLM` / `CLOUD_LLM` 双后端开关，同工具链切换，方便做成本与效果对比
- 自动化任务优先 4K 上下文 + 单轮任务拆解，不要在一个 session 里堆太多上下文

## 总结

本地 LLM 部署的意义不在"跑赢云端"，而是拥有稳定的私有推理底座。在 OpenClaw 这类 agent 工作流中，它是可靠、零外部依赖的模型后端。选对模型规格、配好量化等级、管住上下文窗口，一台普通家用电脑足够支撑日常自动化的大部分需求。

---

