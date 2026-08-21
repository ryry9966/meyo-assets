---
title: 本地 LLM 部署实践：在家用电脑上跑通可被 Agent/MCP 调用的模型
feedId: 34065
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景
在 OpenClaw 的插件、MCP 工具和自动化链路里，很多时候只是想把模型当作本地推理组件：数据不出内网、接口不按 token 计费、调试时能随时重启。但家用电脑的显存、内存和散热决定了不能照搬服务器方案。

## 问题
本地部署的主要矛盾不是“能不能跑”，而是这三件事：
1. 显存不够：加载权重、KV cache 和推理临时张量同时抢显存。
2. 上下文不够：Agent 调用工具后历史很快超过模型默认窗口。
3. 工具调用不稳定：小模型对 function calling 的 schema 遵循度差，输出 JSON 容易非法。

## 做法/步骤
### 1. 模型选型
家用卡优先选 7B/8B 的 GGUF 量化版，例如 Qwen2.5-7B-Instruct 或 Mistral-7B-Instruct，Q4_K_M 是性价比较高的起点。显存估算：权重约 4-5GB，加上 4-8K 上下文的 KV cache 和激活值，8GB 显存可勉强跑，12GB 会舒服一些，16GB 可以尝试 13B/14B Q4，但不建议一上来就上大模型。

### 2. 推理后端
- 单用户、低并发：Ollama 或 llama.cpp server，部署快，适合调试。
- 多 Agent/MCP 并发：建议上 vLLM，因为 continuous batching 和 OpenAI 兼容接口支持更完整。
- 图形化排查：LM Studio 适合快速切换模型和参数，但不建议作为长期常驻服务。

以 Ollama 为例，启动模型后设置环境变量，限制并发和常驻模型数量：
```bash
OLLAMA_NUM_PARALLEL=1
OLLAMA_MAX_LOADED_MODELS=1
```

### 3. 接入 OpenClaw/Agent
把本地模型暴露为 OpenAI 兼容接口后，OpenClaw 或 MCP 工具链只改 `base_url` 和 `api_key` 即可接入。Ollama 的兼容接口通常是 `http://localhost:11434/v1`，`api_key` 填任意非空值。注意不是所有模型都会在首个请求里正确识别 `tools` 参数，建议先用最小工具调用脚本测试：让模型返回固定结构的 JSON，检查是否稳定。

### 4. MCP 工具链适配
对支持 function calling 的模型，直接传 tools schema。若模型不支持，通常有两种办法：
- 在系统提示里写死 JSON 输出格式，并用 few-shot 示例；
- 使用 grammar/constrained decoding 约束输出。

温度设成 0 或接近 0，能明显减少工具调用字段漂移。

## 踩坑点
1. **显存 OOM**：最常见。先不要开长上下文，`num_ctx` 从 4096 起试，GPU offload 层级不要拉满，留 1-2GB 给 KV cache。
2. **CPU 推理很慢**：确认模型实际加载到 GPU。Ollama 可以看日志里 `offloaded layers` 是否接近总层数。
3. **JSON 非法**：小模型容易多输出一个逗号或忽略 `}`。不要只依赖 retry，最好在 prompt 里给一个完整输出示例，并限制 `max_tokens`，避免模型自由发挥。
4. **并发排队**：多 Agent 同时请求时，Ollama 默认会排队，导致工具调用超时。可以先降低并发，或换 vLLM，并限制每个请求的 `max_tokens`。
5. **长上下文爆显存**：KV cache 会随上下文长度非线性增长。工具输出要截断，不要把整个网页或日志塞进历史。

## 可复用建议
- 建立三层模型池：7B/8B 做 Agent 主模型，3B 做分类/摘要，embedding/reranker 独立部署，不要混在一个服务里。
- 所有配置版本化：模型名、量化版本、`num_ctx`、GPU 层数、temperature 写进配置文件，避免“上周还能跑，今天不行”。
- 用脚本做冒烟测试：启动后自动发送一次 tool call 和普通对话，检查返回结构和延迟。
- 监控显存和队列：`nvidia-smi`、`nvtop` 或 Ollama 日志持续观察，发现 OOM 先降上下文，再考虑换模型。

## 总结
在家用电脑上跑大模型并不难，难的是把它稳定地接入 OpenClaw/Agent/MCP 工具链。核心是控制显存、限制上下文、测试工具调用，而不是盲目追求参数规模。先把 7B/8B 量化模型跑稳，再考虑并发和长上下文，是更务实的路线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/d9523c861971fc21.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/f66dbd1e6db78438.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/36f808f7bc509185.png)

