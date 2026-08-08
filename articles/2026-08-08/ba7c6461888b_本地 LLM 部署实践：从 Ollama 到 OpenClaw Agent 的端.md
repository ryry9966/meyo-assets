---
title: 本地 LLM 部署实践：从 Ollama 到 OpenClaw Agent 的端到端指南
feedId: 32105
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景：Agent 场景下为什么需要本地 LLM

在 OpenClaw 的实际使用中，LLM 不只是聊天机器人，更是 Agent 的决策中枢。社区里大量自动化案例依赖 MCP 工具与插件生态，LLM 负责编排工具调用、解析自然语言指令、生成结构化输出。如果每次决策都要走云端 API，三个问题会越来越突出：

1. **延迟不稳定**：Agent 一个动作可能触发多次 LLM 调用，云端抖动会把任务时间拉长数倍。
2. **数据出域焦虑**：工具返回的日志、文件内容、环境变量等敏感信息，一旦上传到第三方 API，安全审核很难通过。
3. **成本不可控**：高频的 Agent 循环很容易吃掉免费额度，规模化后费用陡增。

在家用电脑上部署一个本地 LLM，可以在保留核心能力的同时，让上述问题大幅缓解。本文围绕 Ollama + OpenClaw 的集成，给出一套可复现的工程化实践。

## 问题拆解：桌面硬件的真实约束

普通家用电脑的典型配置是 16 GB 内存、8 核 CPU、无高端显卡。部署 LLM 要解决的几个硬约束：

- 模型必须能装进 6–10 GB 可用内存，意味着 7B–14B 参数级别的模型是上限。
- 推理速度不能影响自动化节奏，单次生成耗时需控制在秒级。
- 工具调用能力足够稳定，不能频繁误解 MCP 函数签名。
- 多 Agent 并发时不会 OOM 或崩溃。

这决定了我们的技术选型：以 GGUF 量化模型配合 Ollama 作为运行时，再用 OpenClaw 的 OpenAI 兼容接入来驱动 Agent。

## 步骤一：Ollama 部署与模型选择

Ollama 在 Linux/macOS/Windows 下均可一键安装，它会自动管理模型文件并暴露 REST API。

```bash
# 安装后启动服务（默认监听 11434 端口）
ollama serve
```

**模型选择**：经过社区多个案例的验证，适合 Agent 场景的模型需要较强的指令遵循与工具调用能力。推荐以下两个起点：

- `qwen2.5:7b-instruct-q4_K_M`（约 4.7 GB）：中文理解好，函数调用较稳定，适合大部分自动化任务。
- `llama3.1:8b-instruct-q4_K_M`（约 4.9 GB）：英文体系 Agent 更成熟，对 MCP 标准工具调用格式兼容不错。

下载并测试：

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```

## 步骤二：接入 OpenClaw 并驱动 MCP 工具

OpenClaw 支持在 `config.yaml` 中配置任意 OpenAI 兼容后端，只需修改 provider 和 api_base：

```yaml
llm:
  provider: openai
  model: qwen2.5:7b-instruct-q4_K_M
  api_base: http://127.0.0.1:11434/v1
  api_key: not-needed
```

重启 OpenClaw 后，Agent 的所有决策都会走本地模型。使用 MCP 工具（如文件系统、shell、web_fetch）时，LLM 会根据系统提示和工具描述生成函数调用。建议在 OpenClaw 的系统提示中强化工具使用的指令，例如：

> “当用户要求操作文件时，必须使用 `filesystem` 工具，严格按照工具定义输出 JSON 格式。”

一个小技巧：可以用 OpenClaw 的模板功能为不同 Agent 预设模型与系统提示，避免每次都改配置。

## 步骤三：关键踩坑与调优

### 1. 上下文长度不足

Ollama 某些版本的默认上下文窗口为 2048 token，这在多轮工具调用中很容易溢出，导致模型忘记前面的工具返回结果。新建一个 Modelfile：

```dockerfile
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER num_ctx 4096
```

```bash
ollama create my-agent-model -f Modelfile
```

然后将 OpenClaw 配置中的模型名改为 `my-agent-model`。

### 2. 内存紧张与量化取舍

`q4_K_M` 是一种平衡方案，如果需要更低内存，可改用 `q4_0`（质量轻微下降）。7B 模型在 q4_0 下仅需约 4 GB 内存，14B 模型 `q4_0` 约 8 GB。实测 14B 的 `q4_0` 在部分复杂工具调用任务上表现优于 7B 的 `q4_K_M`，内存够的话可以权衡精度与速度。

### 3. 并发处理

默认 Ollama 并行请求数有限，若有多个 Agent 同时使用，需要设置环境变量：

```bash
export OLLAMA_NUM_PARALLEL=4
```

但注意内存会成倍增长，4 个并发 7B 模型至少需要 16 GB 空闲内存。可配合 Docker 限制资源：

```yaml
services:
  ollama:
    image: ollama/ollama
    environment:
      - OLLAMA_NUM_PARALLEL=2
    deploy:
      resources:
        limits:
          memory: 12G
```

### 4. 工具调用稳定性问题

小模型在输出函数调用时偶尔会出现格式错误，例如漏掉 `parameters` 字段或产生多余文本。解决方法：

- 在系统提示中给出 1–2 个正确示例（few-shot）。
- 降低 `temperature` 到 0.1–0.3，减少随机性。
- 检查 MCP 工具描述是否过于复杂，尽量将参数说明写得直白简洁。

实测经过上述调整后，本地 7B 模型在常见文件读写、命令执行场景中的成功率可达到 85% 以上，足以承担大量批处理自动化任务。

## 可复用建议

- **镜像加速**：Ollama 模型下载可从国内镜像拉取（如 `ollama.hf-mirror.com` 等），避免中断。
- **持久化配置**：所有 Modelfile 和 Docker Compose 放入项目仓库，团队成员可一键复现相同本地环境。
- **混合部署策略**：将确定性高的工具链（如文件整理、数据处理）交给本地模型，复杂推理或偶尔的开放式问答回退到云端 API，通过 OpenClaw 的 fallback 配置实现。
- **监控与日志**：使用 `ollama logs` 和在 OpenClaw 中开启详细日志，记录每次工具调用的决策过程，便于后续排查误判。

## 总结

把大模型装进家用电脑并不是为了追赶技术潮流，而是从 **延迟可控、数据安全、成本归零** 三个工程目标出发的务实选择。通过 Ollama 量化部署 + OpenClaw 集成 MCP，我们已经跑通了一个完整的本地 Agent 回路。虽然 7B 模型在极端复杂任务上仍有短板，但经过上下文、并发和提示词的细节打磨，它能够覆盖社区中 80% 以上的自动化需求，成为 OpenClaw 工作流里一块可靠的本地引擎。

---

