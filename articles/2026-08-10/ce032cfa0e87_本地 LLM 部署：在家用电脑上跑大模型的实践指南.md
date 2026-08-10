---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 32355
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么要在本地跑 LLM？

如果你正在用 OpenClaw 搭建 Agent、编写 MCP 服务或做自动化流程，大概率已经依赖某个云端大模型 API。但 API 有不可回避的痛点：调用延迟、按量计费、数据出境合规风险，以及调试时频繁的 rate limit。当 Agent 开始接触内部文档、代码库或个人敏感信息时，云端转发就不是最优解。

在家用电脑上部署一个能跑的小参数 LLM，既能满足大部分轻量推理需求，又能保障数据不离开本机，还能与 OpenClaw 的模型后端无缝对接。本指南来自实际工程经验，不夸张，不平铺“一键包”，讲一讲真实可用的部署路径和踩过的坑。

## 硬件要求与现实期待

先泼冷水：不要指望用 4GB 内存的轻薄本跑 70B 模型。当前实用的家庭部署窗口在 **7B–14B 参数**的量化模型。推荐配置：

- **最低**：16GB 内存，无 GPU（纯 CPU 推理）
- **平衡**：16GB 内存 + 6GB 以上显存的 NVIDIA GPU（如 RTX 3060）
- **舒适**：32GB 内存 + 12GB 及以上显存

纯 CPU 推理（使用 llama.cpp 后端）的速度大约 3–8 token/s，够用但交互会稍慢；GPU 推理可达 30+ token/s，接近 API 体验。

## 工具链选型：Ollama + Open WebUI

放弃自行编译 llama.cpp 和折腾 Python 环境，直接使用 **Ollama**。它跨平台、模型库丰富、原生支持量化、提供 OpenAI 兼容 API，非常适合作为 Agent 的本地推理后端。

配合 **Open WebUI**（一个类 ChatGPT 的 Web 界面），可以在浏览器中直接与模型对话、管理模型和查看性能指标，也方便非技术同事一起测试。

## 部署步骤

### 1. 安装 Ollama

Linux:
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
macOS 和 Windows 直接下载官方安装包。Windows 下建议启用 WSL2 或使用 Docker 方式，避免 GPU 透传问题。

### 2. 拉取模型

适合中文 Agent 场景的模型推荐 **Qwen2.5** 系列，平衡指令遵从和工具调用能力：
```bash
ollama pull qwen2.5:7b
```
若需要更强推理，可选 `qwen2.5:14b`（需显存约 9GB，Q4_K_M 量化）。通常会使用 4-bit 量化版本，Ollama 默认拉取的就是量化模型。

### 3. 测试对话与工具调用

先用命令行快速验证：
```bash
ollama run qwen2.5:7b
```
尝试多轮对话，并模拟 function calling 风格的提示，确认模型能输出符合预期的 JSON。

### 4. 部署 Open WebUI（可选）

```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```
启动后访问 `http://localhost:3000`，在设置中选择 Ollama，填入 Ollama 的 API 地址（默认 `http://host.docker.internal:11434`）。如果 Ollama 装在宿主机上，Windows 需特别注意网络配置。

### 5. 接入 OpenClaw Agent

OpenClaw 允许自定义 LLM 后端，只需在模型配置中将 `provider` 设为 `openai`，并修改 base URL：

```yaml
models:
  local-qwen:
    provider: openai
    model: qwen2.5:7b
    api_base: http://localhost:11434/v1
    api_key: ollama
```

之后所有 Agent 的推理请求都会走本地 Ollama 服务。对于 MCP 工具调用场景，确保模型有足够长的 context length，否则会在多次工具调用后截断历史。

## 踩坑记录

**坑 1：上下文长度不足**  
Ollama 默认 `num_ctx` 为 2048，Agent 多次工具调用后对话历史容易超出。必须通过 Modelfile 重新设置：
```
FROM qwen2.5:7b
PARAMETER num_ctx 8192
```
然后 `ollama create my-qwen -f Modelfile`，再使用 `my-qwen` 这个模型名。注意增加上下文会大幅提升内存占用，7B 模型 8192 上下文约需额外 1-2GB 内存。

**坑 2：并发请求导致 OOM**  
当多个 Agent 或一个 Agent 的多个步骤同时请求 Ollama 时，可能会占满内存或显存。解决方案是配置并发队列，或部署多个 Ollama 实例分流不同模型，但这在家庭电脑上不太现实。更务实的做法是限制 Agent 的最大并发度为 1。

**坑 3：Windows GPU 支持**  
Windows 原生 Ollama 支持 GPU，但若使用 Docker 部署 Open WebUI 或其他组件，需配置 CUDA 容器或使用 WSL2 内的 Ollama，避免驱动冲突。推荐直接将 Ollama 安装在宿主机，减少一层抽象。

**坑 4：工具调用输出格式不稳定**  
小模型在生成 function call JSON 时偶尔会多出逗号、缺少引号。可以在 OpenClaw 的提示中强制规范输出格式，并增加简单的后处理正则校验。必要时切换到 `qwen2.5:7b-instruct` 或其他微调过指令遵循的版本。

## 可复用的建议

- **模型选型就锁定 Qwen2.5 或 Mistral 系列**，中文场景 Qwen 更优。始终使用量化版本 Q4_K_M，是性能与质量的甜区。
- **固化你的模型配置**，将所有定制（系统提示、上下文长度、停止词）写入 Modelfile，避免每次手动调整。
- **利用 Ollama 的 API 做健康检查**，Agent 启动前先 `curl http://localhost:11434/api/tags` 确认服务在线。
- **为本地模型单独设计短小精悍的提示词**，不要直接复制云端大模型的提示词模板，小模型对冗余描述更敏感。
- **监控内存与延迟**，可用 `ollama ps` 查看模型资源占用，在 OpenClaw 日志中记录首次 token 延迟，帮助评估前端体验。
- **考虑混合方案**：简单的摘要、分类任务走本地模型，复杂推理或长上下文任务仍可 fallback 到云端 API，通过 OpenClaw 的模型路由策略实现，兼顾效率与成本。

## 总结

在家用电脑上部署 LLM 不再是极客玩具，通过 Ollama 生态已经能够开箱即用地为 Agent 提供隐私、低成本的推理服务。整个栈（Ollama + Open WebUI + OpenClaw）部署时间不会超过半小时，却能为日常自动化工作流带来极大的掌控感。对于敏感数据处理、内网部署和有合规要求的团队，这条路远比依赖外部 API 稳固。

当下 7B 级别的模型已经足够处理大多数工具调用和结构化信息提取，加上合适的量化和工程优化，一台普通的游戏主机甚至迷你 PC 都能成为靠谱的本地推理引擎。

---

