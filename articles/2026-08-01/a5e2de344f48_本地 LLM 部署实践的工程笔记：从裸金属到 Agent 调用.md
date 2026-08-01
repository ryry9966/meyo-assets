---
title: 本地 LLM 部署实践的工程笔记：从裸金属到 Agent 调用
feedId: 31191
source: 综合讨论
publishedAt: 2026-08-01
---

## 为什么要在本地跑大模型

在 OpenClaw 的自动化工作流里，Agent 经常需要调用大模型做信息提取、指令解析或生成回复。完全依赖云端 API 会带来三个痛点：**延迟抖动**、**数据出站焦虑**和**调用频率上限**。对很多纯内部自动化来说，用本地运行的 LLM 替代部分云端任务，既可以降低链路复杂度，也能让 Agent 在断网或内网环境保持可用。

这篇笔记的目标是：在一台普通家用电脑（一台 3060 12 GB 的 Windows 游戏机）上，把本地 LLM 跑起来，并通过 MCP 暴露给 OpenClaw Agent 使用。我们会聚焦在高复现率、低折腾成本的方案上。

## 选型与前置判断

当前适合家用硬件部署的推理框架主要有两个路线：

- **llama.cpp 生态**：纯 CPU/混合 GPU 推理，量化灵活，内存友好。
- **Ollama**：封装了 llama.cpp，带 REST API、模型拉取和简易管理，天然适合做本地 Agent 后端。

考虑到需要让 OpenClaw 通过标准 HTTP 接口调用，我直接选择了 Ollama。它开箱提供 `/api/chat` 兼容格式，省去自己写适配层的工作。

硬件条件参考：
- GPU 显存决定了可以跑多大参数的模型。**7B–8B 的 Q4_K_M 量化模型**需要约 6 GB 显存，13B-14B 同量化等级约需 10–12 GB。
- 如果没有显卡，纯 CPU 推理也能凑合用，但首 token 延迟会高到让 Agent 超时，不适合交互式自动化。

## 部署步骤

**1. 安装 Ollama**

官网下载 Windows 安装包，默认安装后会在系统托盘中常驻一个服务。安装完成打开终端，确认可用：

```bash
ollama --version
```

**2. 拉取模型**

这里首选 Meta 的 Llama 3.1 8B 或 Mistral 7B（支持函数调用微调的版本对 Agent 更友好）。直接拉取量化版：

```bash
ollama pull llama3.1:8b-instruct-q4_K_M
```

注意 tag 写明量化和指令版本，避免无意中拉取未量化原版爆显存。

**3. 验证推理并调整上下文长度**

默认上下文窗口只有 2048，做较复杂的 Agent 任务时容易截断。可以通过创建自定义 Modelfile 或参数传递增大上下文：

```
# Modelfile
FROM llama3.1:8b-instruct-q4_K_M
PARAMETER num_ctx 4096
```

然后创建并测试：

```bash
ollama create my-agent-model -f Modelfile
ollama run my-agent-model
```

上下文越大，显存占用会快速上升，建议在自己机器上逐步探测上限。3060 12 GB 设 4096 基本稳定。

**4. 接入 OpenClaw MCP**

Ollama 的 API 端点默认为 `http://localhost:11434`，与 OpenAI 的 chat completions 结构高度相似。OpenClaw 的 MCP 插件可以直接适配，配置一个 “generic-openai” 类型后端，将 base_url 指到 Ollama。例如在你的 MCP 配置中：

```json
{
  "mcpServers": {
    "local-llm": {
      "command": "...",
      "args": [...],
      "env": {
        "OPENAI_BASE_URL": "http://localhost:11434/v1",
        "OPENAI_API_KEY": "ollama"
      }
    }
  }
}
```

具体字段因 OpenClaw 插件实现而异，核心是把流量导到本地的 `/v1/chat/completions` 即可。

**5. Agent 调用与任务测试**

用一个简单的自动化做验证：“阅读剪贴板内容，提取其中的会议要点并用中文总结”。Agent 将被配置为使用本地模型，观察端到端延迟和输出质量。

## 踩坑与解决

| 问题 | 现象 | 处理 |
|---|---|---|
| 显存溢出 | 加载模型时报 CUDA out of memory | 降低量化等级到 Q3_K_M 或限制上下文至 2048，改用 7B 而非 13B |
| 模型拒绝执行 | Agent 要求提取文本，模型输出 “I can’t help with that” | 指令微调版本（instruct）比 base 版更听话；在 system prompt 明确要求“仅执行文本任务” |
| 输出格式不符合 JSON | Agent 需要结构化输出，但模型返回自然语言 | 使用 Ollama 的 `format: json` 模式，或者在提示词末尾强制加“仅返回 JSON，不要添加解释” |
| 纯 CPU 下 Agent 超时 | Windows 上默认超时可能 60s，首 token 延迟超时 | 将 Agent 的请求超时调大至 120s，并降低模型到 3B 或 TinyLlama |
| 上下文不足 | 多轮对话中较早信息被遗忘 | 在配置中明确设定 num_ctx，并让 Agent 实现摘要在对话中主动压缩历史 |

## 可复用建议

1. **模型选型固定化**：为团队固定一个量化模型名称，避免不同人拉取不同版本导致行为不一致。
2. **隔离测试**：先在命令行或 Open WebUI 中单独测试模型对提示词的服从性，再接入 Agent。
3. **限制上下文就是省资源**：大多数自动化任务其实不需要 8k 以上的上下文，2048–4096 能减少 30%+ 的显存占用。
4. **预热模型**：Ollama 在首次请求时才会加载模型，导致第一次调用极慢。在 Agent 启动时加入一个 `/api/generate` 的 healthcheck 触发加载。
5. **兜底策略**：本地模型总有卡壳的时候，可以在 Agent 工作流里设置 fallback 到云端小模型，或返回固定错误提示，防止自动化中断。

## 总结

在家用电脑上部署本地 LLM 并把它变成 Agent 的后端，工程难度已经比一年前低了很多。Ollama 的标准化接口和 OpenClaw MCP 的灵活适配，让整个过程可以在几十分钟内打通。关键不在于盲目追求大参数量，而是根据实际任务把**量化方案、上下文大小和提示词约束**调到一个可用的均衡点。

对于大量文本提取、内网数据处理和低延迟影响的自动化场景，这套部署模式完全可以替代部分云端 LLM 调用，极大减少对外部服务的依赖。记录一些硬核调试数据后，也能更清晰地知道什么时候该用本地模型，什么时候该走云端。

---

