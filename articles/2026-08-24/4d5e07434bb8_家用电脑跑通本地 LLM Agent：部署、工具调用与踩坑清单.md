---
title: 家用电脑跑通本地 LLM Agent：部署、工具调用与踩坑清单
feedId: 34517
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

有些场景不适合把 Agent 的推理请求发到外部 API：插件要处理本地文件、自动化脚本里包含敏感配置、内网环境不能出外网，或者只是不想调试时受限流和账单影响。本地 LLM 是补充，不是替代。

对 OpenClaw / MCP / 插件这类链路来说，本地 LLM 的目标不是“能聊天”，而是“能稳定输出可解析的工具调用”。因此部署的重点不是聊天界面，而是一个 OpenAI 兼容、可被 Agent 直接调用的本地推理服务。

## 问题

家用电脑跑 LLM 主要卡在三点：

1. **显存不够**：大模型放不进去，量化后能力会下降。
2. **工具调用不稳定**：模型容易输出自然语言而非 JSON，导致 MCP 解析失败。
3. **推理并发低**：Agent 并行调用工具时容易排队超时。

这三个问题都不是换一个更强模型就能完全解决的，需要从部署参数、模型选型和接入方式上一起控制。

## 做法 / 步骤

### 1. 先按显存选模型

Q4_K_M 量化后的体积大致是：7B-9B 约 4-6GB，13B-14B 约 8-10GB，32B 约 20-22GB。

- 8GB 显存：优先 7B/8B 模型。
- 16GB 显存：优先 14B 模型。
- 24GB 显存：可以跑 32B Q4 或 14B 高量化。

CPU 推理可以跑通，但 Agent 多轮工具调用会慢到超时，不建议作为主力。

模型优先选带 instruct 且对 function calling 友好的版本，例如 `qwen2.5:7b/14b/32b-instruct`、`llama3.1:8b-instruct` 等。不要拿基础模型直接接工具。

### 2. 用 Ollama 快速起服务

Linux 下 Docker 运行：

```bash
docker run -d --gpus all -p 11434:11434 \
  -e OLLAMA_HOST=0.0.0.0 \
  -e OLLAMA_NUM_PARALLEL=1 \
  -e OLLAMA_CONTEXT_LENGTH=8192 \
  -v ollama:/root/.ollama \
  ollama/ollama:latest
```

下载模型：

```bash
docker exec -it <container> ollama pull qwen2.5:14b-instruct-q4_K_M
```

如果下载慢，可以配置镜像或使用 `HF_ENDPOINT` 导入 GGUF。Ollama 默认暴露 `http://127.0.0.1:11434/v1`，OpenAI 兼容客户端把 `base_url` 指过去即可，`api_key` 可任意填写。

### 3. 先用 curl 验证工具调用

不要一上来就接 Agent。先用一个最小工具定义测模型是否返回可解析 JSON：

```json
{
  "model": "qwen2.5:14b-instruct-q4_K_M",
  "messages": [{"role":"user","content":"查一下北京天气"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "Get weather for a city",
      "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
      }
    }
  }],
  "temperature": 0
}
```

如果返回的是自然语言而不是 `tool_calls`，说明模板或模型能力不匹配。先换模型或调整提示词，再接入插件。

### 4. 接入 OpenClaw / MCP

本地服务作为 OpenAI 兼容端点接入。MCP 工具较多时，先只暴露 3-5 个核心工具，字段和 description 尽量短。上下文由 `OLLAMA_CONTEXT_LENGTH` 控制，建议从 8192 开始，跑通后再按需调大。

## 踩坑点

- **工具调用输出不稳定**：模型会先说“我来调用工具”，然后输出非 JSON 内容。温度设 0 或 0.1，不要加过多角色扮演 system prompt。必要时在客户端做一层解析兜底：从输出中提取 JSON 块，失败则重试一次。
- **OOM 和 KV cache 爆显存**：上下文长度不是越大越好。Ollama 默认上下文可能偏小，但调太大容易 OOM。设置 `OLLAMA_CONTEXT_LENGTH` 后，观察启动日志和显存占用。
- **多并发排队**：本地推理并发低，Agent 并行调用多个工具时容易超时。服务端设 `OLLAMA_NUM_PARALLEL=1`，客户端把单次请求超时放大到 120s，并限制并行工具数。
- **大工具 schema 挤占上下文**：几十个 MCP 工具的全量描述可能占掉 2-4K token，小模型还没推理就乱了。按任务裁剪工具，只保留必要参数。
- **下载和版本不一致**：固定 commit/量化版本，记录 sha256。不要同时混用不同量化和模板的同一模型。

## 可复用建议

- 把本地 LLM 单独做成一个服务，不嵌进 Agent 进程。以后换模型、换量化、换推理后端都不影响插件侧。
- 新建一个 `local-llm` 目录，里面放 `docker-compose.yml`、模型清单、curl 测试脚本和一组回归指令。每次升级模型都先跑回归。
- 对工具调用结果做 post-validation：字段缺失、类型错误、非法 JSON 时让 Agent 重试，或直接回退到规则逻辑。
- Mac 用户可优先用 Ollama 的 Metal 后端，但同样建议固定上下文和并行数。
- 如果对吞吐有更高要求，再考虑 `llama.cpp server` 或 vLLM；日常插件调试，Ollama 足够。

## 总结

家用电脑跑本地 LLM 的可行目标，是做一个稳定的 OpenAI 兼容推理端点，而不是追求最大模型。先把显存、量化、上下文、并行数和工具 schema 控制住，再用最小测试验证 `tool_calls`，最后接 OpenClaw / MCP。这个顺序能少踩很多坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2d7f93d6b0499a81.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1b54ff536d24de5e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/0da1cf27286e0971.png)

