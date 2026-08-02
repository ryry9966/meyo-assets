---
title: 在家用电脑跑大模型：面向 OpenClaw 插件与 Agent 的本地 LLM 部署实践
feedId: 31330
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：为什么要在本地跑 LLM？

对已经接入 MCP 服务、正在写自定义 OpenClaw 插件的人来说，云 API 虽然方便，但几个痛点越来越痛：

- **延迟不稳定**：Agent 链式调用时，每次调用都跨公网，一旦链路长，整个自动化脚本的执行时间会翻倍。
- **敏感数据风险**：通过 MCP 工具拉取的本地文件、数据库内容，不太想全部交给第三方 API。
- **成本不可预测**：调试 Agent 行为或跑批量任务时，很容易因为输入太长消耗大量 token，月初就收到账单预警。
- **上下文可控性弱**：云端 API 常有 token 上限和速率限制，对于需要长时间、多轮对话或无限循环推理（ReAct 类 Agent）的场景不够用。

因此，在本地部署一个可控、可调、无联网烦恼的 LLM，就成了一件值得认真做一次、然后长期复用的工程任务。

这篇实践指南不谈论“效果能不能追上 GPT-4”，只关注：**如何在普通家用电脑上跑通一个可以为 OpenClaw 代理提供推理能力的本地模型，并把坑点讲清楚。**

---

## 问题拆解：我们到底要解决什么？

目标场景很具体：OpenClaw 作为 Agent 运行时，通过 MCP 协议与本地工具交互（文件系统、数据库、浏览器、CLI）。我们需要一个本地 LLM，做到：

1. **在消费级硬件上可运行**（8–16 GB 显存，或纯 CPU 推理也能动）；
2. **与 OpenAI 兼容的 API 接口**，方便直接配置到 OpenClaw 的 provider；
3. **延迟可接受**，单次简单对话 < 3 秒，工具调用链能进行；
4. **支持 function calling / tool use**，因为 Agent 必须能返回结构化的工具调用意图。

在这个设定下，我们选择了 **Ollama + qwen2.5 系列模型** 作为主力部署方案，顺便拆一下实测中踩到的坑。

---

## 做法与步骤

### 1. 环境选择与初始化

硬件参考：一台 Ryzen 7 5800X + RTX 3060 12 GB 显存、32 GB 内存的台式机。

安装 Ollama：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

确保 Docker 不在同一台机器上占用 GPU，否则会出现显存碎片问题（后面会讲）。

### 2. 先挑模型再做量化决策

不要一上来就拉最大的。先确定用例：

- 简单分类、摘要、意图识别：`qwen2.5:7b-instruct-q5_K_M`（约 4.5 GB）
- 工具调用、多步推理：`qwen2.5:14b-instruct-q4_K_M`（约 8.5 GB）
- 复杂 Agent 或需要长上下文：`qwen2.5:32b-instruct-q4_K_M`（约 19 GB，可用但会 spill 到内存）

使用 `ollama pull qwen2.5:14b-instruct-q4_K_M` 这种精确 tag，避免默认下载 `q4_0`。

### 3. 启动与调参

创建 `Modelfile`：

```
FROM qwen2.5:14b-instruct-q4_K_M
PARAMETER temperature 0.7
PARAMETER num_ctx 8192
PARAMETER num_predict 2048
```

然后 `ollama create my-agent-model -f Modelfile`。

关键点是 `num_ctx` 不能给太大。第一次我设了 32768，导致内存狂涨，甚至引发系统 OOM。家用机给 8192 是稳妥值。

### 4. 提供 OpenAI 兼容端点

Ollama 默认在 `http://localhost:11434/v1` 提供 OpenAI 兼容 API。在 OpenClaw 的 `provider.yaml` 里配置：

```yaml
llm:
  provider: openai
  base_url: http://localhost:11434/v1
  api_key: ollama
  model: my-agent-model
```

然后重启 OpenClaw，它就能用本地模型执行 Agent 规划与工具选择了。

---

## 踩坑点汇总（含排障思路）

### 坑 1：显存不够，模型落入 CPU推理

即使显存看起来够（12 GB 拉 8.5 GB 模型），实际推理时会超出，因为 KV cache 会按上下文长度动态分配。以 14b q4_K_M 模型为例，`num_ctx=8192` 时的 KV cache 约 2 GB 左右，加上模型 8.5 GB，总显存需求接近 11 GB。如果桌面环境、浏览器再占用一点显存，就会触发 CUDA OOM，推理速度从 40 tokens/s 骤降到 5 tokens/s。

**解法**：
- 用 `nvidia-smi` 监控并关闭无用的图形程序。
- 明确设置 `OLLAMA_GPU_LAYERS` 环境变量，限制层数，让一部分推理落到 CPU，换取稳定速度。
- 或者直接选择 7b 模型，显存压力瞬间消失。

### 坑 2：多轮 Agent 调用时上下文爆炸

Agent 工作流（思考 → 观察 → 反思 → 行动）会在对话历史中积累大量工具调用返回的结果。如果 MCP 工具返回的网页内容、数据库查询结果很长，很快填满 `num_ctx`，模型就开始胡言乱语，或返回不完整的 JSON。

**解法**：
- 在 OpenClaw 的 skill 或 tool 描述里加入裁剪指令，例如限制返回前 1500 个字符。
- 开启 Ollama 的 `num_ctx` 截断策略，配合 `ollama/server` 的 `--ctx-size` 启动参数。
- 设计一个历史管理工具（MCP 插件），对超过一定长度的消息作摘要。

### 坑 3：工具调用格式不稳定

有些本地模型对 OpenAI 格式的 tool_calls 输出不够稳定，可能出现不闭合的 JSON，或函数名错误。qwen2.5:14b 表现尚可，但 7b 版本偶尔出错。

**建议**：
- 在 system prompt 里明确给出函数返回的 JSON Schema，并加上一个强约束：“只输出有效 JSON，不输出额外文本”。
- 在 OpenClaw 的插件侧做好防御：对模型返回的工具调用做 JSON 解析，失败就退回到默认文本回复，避免整个工作流中断。

### 坑 4：重启后模型消失或版本变化

如果不指定 tag，`ollama pull` 会拉取 `latest`，某天模型更新后行为可能不一致，导致稳定运行的 Agent 突然失效。

**建议**：
- 始终用精确 tag。
- 将 Modelfile 和模型清单纳入版本控制，并用脚本固化部署流程。

---

## 可复用建议

- **按场景分层用模型**：本地跑小模型做意图路由、格式校验；真正困难的推理才调用云端强模型。OpenClaw 支持多 provider，可以做这种“路由器”。
- **为工具编写者建立模型兼容性表**：MCP 工具作者应在文档中标注测试过的本地模型与所需最低量化，降低集成成本。
- **监控先行**：部署后立刻接上 Prometheus + Grafana，盯着 `ollama_processing_tokens_per_second` 和显存水位，避免凌晨自动化任务把机器打崩。
- **建立回退链**：如果本地模型 OOM 或掉速，OpenClaw 应能自动切换到备用 provider（比如一个低价云 API），而不是直接报错。

---

## 总结

在家用电脑上稳定运行能给 Agent 干活的大模型，难点不在安装，而在**资源管理、上下文控制和输出稳定性**。通过合适的量化模型、显存策略和工具返回预处理，完全可以让一个 14b 模型支撑起日常的 OpenClaw 插件调用与 MCP 自动化任务。

这套部署一旦跑通，收获的不只是数据留在本地，还有随时可改、不受 rate limit 约束的调试体验。对持续迭代 Agent 工作流的人来说，这才是真正的价值。

---

