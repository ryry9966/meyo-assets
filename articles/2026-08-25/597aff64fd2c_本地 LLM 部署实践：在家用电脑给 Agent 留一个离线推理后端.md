---
title: 本地 LLM 部署实践：在家用电脑给 Agent 留一个离线推理后端
feedId: 34630
source: 综合讨论
publishedAt: 2026-08-25
---

OpenClaw 的 Agent、MCP 和插件大多默认接云端模型，但很多自动化任务其实不需要最强模型，而是需要一个稳定、可控、不把数据发出去的本地推理后端。家用电脑只要有一张中端显卡，完全能承载 7B–14B 量化模型。这里不讨论“本地模型取代云端”，只讲怎么让它稳定接入 OpenClaw 的工具调用链路。

## 背景与问题

家用场景的核心约束不是“能不能跑”，而是“跑起来后能不能稳定执行工具调用”。常见问题包括：显存不足、量化损失导致 JSON 输出断裂、上下文窗口太小、API 格式不兼容，以及 Agent 多轮工具循环下首 token 太慢导致超时。

所以本地部署的目标应该很明确：做一个离线兜底、私密任务、插件调试和批量自动化可用的推理后端，而不是追求跑分。

## 做法与步骤

**1. 硬件与模型匹配**

先看显存。8GB 显存适合 7B Q4_K_M；12GB 可跑 14B Q4；16GB 可尝试 14B Q5 或 32B Q3/Q4。但 Agent 场景不建议 Q3，工具调用容易碎。32GB 内存可以做 CPU offload，但首 token 会明显变慢，多轮 Agent 容易超时，只适合做一次性补全或简单分类。

**2. 模型选型**

优先选有工具调用能力的 instruct 版本，例如 Qwen2.5/3、Llama 3.1/3.2、Mistral 等。不要选 base 模型，也不要选只做过通用聊天对齐、没有可靠 function calling 的模型。GGUF 量化格式更适合家用电脑，Q4_K_M 或 Q5_K_M 是平衡点。

**3. 推理后端**

家用单卡最省事的是 Ollama。它提供 OpenAI 兼容的 `/v1/chat/completions`，可以直接被 OpenClaw 当本地 OpenAI API 使用。需要更细控制时再上 llama.cpp 或 LM Studio；vLLM 对单卡家用没有明显收益，反而多一层显存管理。

**4. 加载与运行**

导入模型后先跑通：
```
ollama run qwen2.5:14b-instruct-q4_K_M
```
然后用 `ollama ps` 查看显存占用，避免加载后才发现超出显存。

**5. 验证工具调用**

这是最关键的一步。不要直接接入 Agent，先用 `curl` 发一个带 `tools` 的请求，确认返回结构里出现 `tool_calls`，而不是把工具说明当成普通文本输出。如果这一步不通过，接入 OpenClaw 后只会得到大量解析错误。

**6. 接入 OpenClaw**

在 OpenClaw 的 OpenAI 兼容配置里把 `base_url` 指向：
```
http://127.0.0.1:11434/v1
```
`api_key` 可以随意填，例如 `local`；模型名使用本地模型名称。MCP server 仍然只负责工具执行，不要把模型推理塞进 MCP，否则会把本地资源吃满。

**7. 参数设置**

温度调到 0.1–0.3，关闭极端采样参数，限制 `max_tokens`，系统提示词保持简洁。给本地模型单独写 system prompt，要求只做单步工具调用，不要让它自由发挥。

## 踩坑点

- **量化模型 JSON 截断**：工具参数写到一半断裂，通常是因为温度过高或量化等级过低。先降温度，再升量化版本。
- **上下文不够**：Ollama 默认上下文约 4K，Agent 多轮工具循环可能不够。可设置 `num_ctx 8192`，但会明显增加显存占用。
- **KV cache 爆显存**：上下文越长，KV cache 越大。家用卡不要盲目加到 16K 以上，先跑 8K 验证稳定性。
- **多实例同时加载**：不要同时加载多个大模型实例，显存碎片会拖垮推理速度。
- **Windows 驱动问题**：优先用 WSL2 或 Docker，少用原生安装，避免共享内存和驱动兼容问题。
- **API 格式不兼容**：有些客户端会发送新版 `responses` 或扩展 `tools` 格式，本地模型只支持基础 `chat/completions` 时需要降级或关闭部分选项。

## 可复用建议

固定一个本地测试栈：Ollama + Open WebUI + OpenClaw。先用 curl 跑通最小闭环：用户指令 → 本地 LLM 返回 `tool_call` → MCP 执行 → 结果回传。保留一个云端小模型作为对照，本地模型解析失败时自动降级。锁定模型版本，升级前先在临时实例验证工具调用是否正常。

## 总结

家用电脑跑本地 LLM 不是替代云端模型，而是给 OpenClaw 的 Agent、MCP 和自动化任务提供一个可控的离线后端。选对模型、验证工具调用、控制上下文和量化格式，比盲目追求参数量更重要。跑通一次最小闭环，后续就可以把本地推理当作稳定组件复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/812bb00a4dc68c3c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d1b7d2ff95df8502.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/6c7ed70d97d3e83d.png)

