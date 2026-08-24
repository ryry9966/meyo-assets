---
title: 家用电脑跑本地 LLM：从 Ollama 到 OpenClaw Agent 的实践记录
feedId: 34495
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在折腾 OpenClaw、MCP 和插件自动化时，很多任务会依赖 LLM 做规划、工具调用或结构化输出。云 API 虽然省事，但有成本、延迟和隐私问题，尤其是调试阶段频繁调用、日志里混入敏感数据，或者需要离线运行的场景。家用电脑现在已经有能力跑 7B～32B 级别的量化模型，把推理下沉到本地，对 Agent 开发来说是一个可选项。

## 问题

家用电脑跑 LLM 的主要约束不是“能不能跑”，而是：

- 显存/内存有限，模型尺寸和上下文长度要精打细算。
- Agent 场景需要模型具备稳定的 function calling / tool use 能力，不是所有小模型都合格。
- 本地推理服务要能兼容 OpenAI API，才能被 OpenClaw、MCP 或插件直接调用。
- 出错时排查链路比云 API 多一层：模型、推理后端、驱动、API 兼容层。

## 做法/步骤

**1. 先盘硬件，再选模型**

家用机常见配置：

- 8GB 显存：7B/8B 模型 Q4_K_M 或 Q5_K_M。
- 12GB～16GB 显存：14B 模型 Q4_K_M，或 7B/8B 模型配上较长上下文。
- 24GB 显存：32B 模型 Q4_K_M，或 14B 模型跑更长上下文和更高并发。

个人推荐从 Qwen2.5-7B-Instruct 或 Llama-3.1-8B-Instruct 开始，这两个系列对工具调用支持较好，社区反馈也比较稳定。显存充裕再上 Qwen2.5-14B/32B-Instruct。不要一上来跑 70B，调试起来会很痛苦。

**2. 用 Ollama 作为推理后端**

Ollama 安装简单，自带 OpenAI 兼容接口，适合工程化使用。安装后执行：

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
ollama serve
```

建议显式设置环境变量，避免默认值不满足 Agent 并发需求：

```bash
export OLLAMA_HOST=0.0.0.0:11434
export OLLAMA_NUM_PARALLEL=2
export OLLAMA_CONTEXT_LENGTH=8192
```

`OLLAMA_CONTEXT_LENGTH` 必须根据显存调整。7B Q4 模型在 8GB 显存上设 4096～8192 比较稳，过大容易 OOM。

**3. 验证 OpenAI 兼容接口**

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:7b-instruct-q4_K_M",
    "messages": [{"role":"user","content":"ping"}],
    "temperature": 0
  }'
```

能正常返回就说明基础链路通了。

**4. 接入 OpenClaw / Agent / MCP**

把本地 Ollama 当作 OpenAI 兼容服务填进配置：

- base_url: `http://127.0.0.1:11434/v1`
- api_key: 随便填，如 `local`
- model: 你拉取的模型名

注意：Agent 工具调用对模型要求比普通对话高。建议在配置里把 `temperature` 设为 0 或接近 0，减少结构输出漂移。如果模型不支持原生 function calling，可以改用 prompt 约束输出 JSON，但稳定性会下降，不建议作为主链路。

**5. 性能调优**

- 确认 GPU 被使用：`nvidia-smi` 在推理时应有进程占用显存。
- 如果推理很慢，先看是否跑到 CPU 上。Ollama 在 Windows/WSL2 下有时需要装好 CUDA 和 GPU 驱动。
- 上下文不要开太大。很多 Agent 任务默认带很长的 system prompt，本地模型在长上下文下 prefill 会明显变慢。
- 如果需要高频调用，限制并发或增加 `OLLAMA_NUM_PARALLEL`，避免多个 Agent 同时请求时排队。

## 踩坑点

1. **OOM**：最常见的坑。模型加载时显存够，一跑长上下文就爆。解决方法是降低上下文长度、换更低量化或更小模型。
2. **工具调用不稳定**：小模型容易出现漏参数、输出非 JSON、幻觉工具名。选择明确支持 tools 的 Instruct 模型，并在提示词里给出清晰工具描述。
3. **OpenAI 兼容层差异**：Ollama 的 `/v1` 接口并非 100% 覆盖 OpenAI 字段，比如部分 `response_format` 或 `tools` 高级参数可能被忽略。接入前先做最小化工具调用测试。
4. **并发阻塞**：单卡家用机同时跑多个 Agent 请求会排队。如果插件或 MCP 触发重试，可能放大延迟。
5. **模型下载与磁盘占用**：Q4 量化模型动辄 4～8GB，注意磁盘空间，且不同版本模型名不同，升级时容易混用。

## 可复用建议

- 固定模型版本和量化格式，写在配置里，不要用 `latest` 标签。
- 本地模型只负责明确能力范围内的任务，复杂规划仍可保留云模型兜底。
- 为 Agent 做一个最小化工具调用测试集，换模型或升级后先跑一遍。
- 监控显存和请求队列：`nvidia-smi`、`ollama ps`、日志里看排队情况。
- 如果要在局域网内供其他机器调用，Ollama 需监听 `0.0.0.0`，并做好访问控制。

## 总结

家用电脑跑本地 LLM 已经不是玩具，7B～32B 量化模型配合 Ollama 可以支撑不少 OpenClaw、MCP、插件自动化的调试和轻量生产场景。关键是把硬件显存、模型能力和 Agent 工具调用三件事对齐，避免用云 API 的习惯无脑套到本地。落地时保留云端 fallback，能让本地推理真正成为工程链路里可维护的一环，而不是一次性的性能实验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/9b32c63b2479ed55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/bc21c46ea8f3f92c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/02921c42d9b8080c.png)

