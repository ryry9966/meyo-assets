---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 34185
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

对常年在 OpenClaw、MCP、插件和自动化任务里折腾的人来说，本地 LLM 不是一个“取代云端 API”的口号，而是一种很实际的工程选择。

典型的场景是：任务里夹着敏感配置、内网数据或大量自动化循环，不太想每次调用都上云；或者需要离线跑批，希望减少延迟和公网波动；也可能是为了在本地复现一条 Agent 链路时，让环境变量可控、可调试。

家用电脑的限制也摆在那里：显存有限、推理速度一般、小模型工具调用不稳定。实践目标不应该是跑最大的模型，而是跑一个“能稳定完成当前工作”的最小模型。

## 问题

在 OpenClaw/Agent 体系里接本地模型，最容易出问题的不是“跑不起来”，而是“跑起来之后不好用”。

常见问题包括：

- 模型选错版本，拉的是 base 模型，接给 Agent 后输出像补全，不按指令来。
- 显存不够，Ollama 或 llama.cpp 自动把一部分层放到 CPU，速度骤降但还能跑，容易被误判为“模型不行”。
- 默认上下文长度太小，多轮工具调用很快被截断。
- 小模型 function calling 能力一般，MCP 工具名、参数名经常少一个、错一个，导致整条链路失败。
- OpenClaw 侧仍用云端模型参数，比如 `thinking`、`reasoning_effort` 或严格 JSON Schema，本地服务端不支持直接报错。

## 做法 / 步骤

### 1. 先评估硬件，设定可用标准

按显存粗分：

- 8GB 显存：优先跑 7B–8B 的 Q4_K_M 量化，例如 Llama-3.1-8B-Instruct。
- 12GB 显存：可以跑 14B Q4_K_M，速度尚可。
- 没有独显或显存很少：可以跑 CPU + 内存，但建议只跑 7B 以下，并且不要抱太高预期。

交互和 Agent 循环的速度可以这样判断：`>8 tok/s` 基本可用，`>20 tok/s` 体验较好。可以用 `ollama run` 后连续对话测试。

### 2. 选推理后端

个人建议：

- **Ollama**：最省事，安装、拉模型、提供 OpenAI 兼容 API 一条龙，适合快速接入 OpenClaw。
- **llama.cpp / llama-server**：适合需要精细控制 `--n-gpu-layers`、`--ctx-size`、多显卡 offload 的人。
- **LM Studio**：图形化操作，适合调试，但自动化和服务化不如前两者方便。

示例：Ollama 拉取一个本地模型：

```bash
ollama pull qwen2.5:14b-instruct-q4_K_M
ollama list
ollama run qwen2.5:14b-instruct-q4_K_M
```

如果用 llama-server，可以这样启动：

```bash
llama-server -m model.gguf --ctx-size 8192 --n-gpu-layers 99 --port 8080
```

`--n-gpu-layers` 不是越多越好。超过显存后会明显回退到 CPU，速度可能从 20 tok/s 掉到 3 tok/s。

### 3. 选本地模型

只选 **instruct 版本**，并且优先选择明确支持 tool calling / function calling 的模型。对家用场景，建议从 7B–14B 开始，Q4_K_M 量化通常是个平衡点。

给 OpenClaw 用，我比较倾向这些类型：

- Qwen2.5 系列：指令、中文、工具调用整体稳定。
- Qwen2.5-Coder 系列：插件、代码生成、结构化输出更稳。
- Llama-3.1-8B/14B-Instruct：英文和基础工具调用不错。
- Mistral-Nemo 或类似 12B：在部分设备上速度与能力平衡较好。

不建议一上来就跑 30B、70B。家用显卡跑 70B 通常要么 OOM，要么速度太慢，无法支持多步 Agent 循环。

### 4. 接入 OpenClaw

本地推理服务跑起来后，OpenClaw 一般不需要写插件，当成 OpenAI 兼容服务即可。

配置方向可以类似这样：

```yaml
provider: openai-compatible
base_url: http://127.0.0.1:11434/v1
api_key: ollama
model: qwen2.5:14b-instruct-q4_K_M
temperature: 0.1
```

不同版本的 OpenClaw 配置字段可能略有差异，但核心就是三点：

- `base_url` 指向本地服务，注意保留 `/v1`。
- `api_key` 可填任意串，Ollama 不强制。
- `model` 必须和 `ollama list` 里的 tag 完全一致，否则返回 404。

接入前先用 curl 验证：

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:14b-instruct-q4_K_M","messages":[{"role":"user","content":"ping"}]}'
```

### 5. MCP / 工具调用单独验证

本地模型接入 Agent 后，不要直接跑复杂任务。先单独测工具调用：给它一个需要调用某个 MCP 工具的 prompt，检查返回的 `tool_call` 是否为合法 JSON、工具名是否正确、参数名是否齐全。

如果发现工具调用不稳，常见缓解办法：

- 减少单轮可用工具数量，保留 2–4 个最必要的工具。
- 把工具 schema 写短，枚举值写死，不要给模型太多自由文本。
- 在 system prompt 里给一个完整工具调用示例。
- 温度调低到 `0.0–0.2`，关闭不必要的采样参数。

## 踩坑点

1. **默认上下文不够**  
   Ollama 默认 `num_ctx` 较小，长任务很快被截断。建议在 Modelfile 或请求 `options` 里显式调到 8192–16384，但越高显存占用越大。

2. **显存被图形界面吃掉**  
   Windows 桌面、浏览器、IDE 都可能占 1–3GB 显存。启动本地模型前，先用 `nvidia-smi` 查一下剩余显存。Ollama 的 `ollama ps` 里如果 `PROCESSOR` 显示混着 CPU/GPU，说明你有层被回退到 CPU，需要减少模型体积或释放显存。

3. **量化不是越小越好**  
   Q2、Q3 虽然小，但指令遵循和工具调用下降明显。对 Agent 场景，Q4_K_M 是底线，条件允许可用 Q6_K。

4. **OpenAI SDK 参数未必兼容**  
   `response_format={"type":"json_schema"}`、`stream_options`、`thinking` 等参数在本地服务端不一定支持。接入 OpenClaw 时常会因为多传了一个参数直接 400。原则：先 curl 测试，再改配置。

5. **模型名写错导致静默失败**  
   配置里写 `qwen2.5:14b` 而本地只有 `qwen2.5:14b-instruct-q4_K_M`，可能导致每次启动都拉不到模型。一定用 `ollama list` 核对完整 tag。

6. **CPU 跑大模型误判为可用**  
   有的 14B 模型在 CPU 上能加载，但只有 2–3 tok/s。OpenClaw 那边可能表现为超时、工具调用迟迟不返回。不是模型坏了，是速度不达标。

## 可复用建议

- 固定一个“本地默认模型”和对应参数字段，不要频繁换模型。先让一条最小链路稳定跑通。
- 对 Agent 任务，温度尽量低；重复惩罚不要设太高，否则工具调用会变得很碎。
- 保留一个云端 API 作为 fallback。OpenClaw 里最好通过配置切换，而不是在代码里写死本地模型。
- 监控三个数：首 token 延迟、`tok/s`、显存占用。用 `ollama ps` 和 `nvidia-smi` 就能看，不必上复杂监控。
- 如果主要跑中文自动化、插件和 MCP，优先选 Qwen2.5 或同级别经过工具调用训练的 instruct 模型，稳定性要求高于单纯聊天能力。

## 总结

本地 LLM 在家用电脑上的定位，是承接低风险、长尾、敏感或离线任务，而不是替代云端大模型。

建议从 `7B/14B + Q4_K_M + Ollama` 开始，先把 OpenClaw 的 OpenAI 兼容 API 打通，再单独验证工具调用。处理显存、上下文、模型 tag 和错误参数，比换更贵的硬件更能提升成功率。

本地模型不可靠时，留一条云端回退路径。工程化使用，最重要的不是“完全离开云”，而是该上的上，该本地的本地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/06066515c0000808.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/2e8d2b92893e76f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4dbd4a27806b2e2a.png)

