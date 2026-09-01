---
title: 本地 LLM 部署实践：让 Agent 在家用电脑上离线跑起来
feedId: 35746
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

Agent 自动化有一个隐藏成本：token。一个跑 MCP 工具链的循环任务，一轮下来几十次工具调用、几十万 token 很常见。放在云端 API 上，费用和隐私都是问题——尤其任务涉及本地文件、内网服务时，数据出域本身就让人犹豫。把模型搬回本地是自然选择：一次硬件投入，换 unlimited 试错额度和"数据不出门"的安心。

## 问题

家用环境跑本地模型，核心矛盾有三个：

1. 显存/内存预算有限，模型尺寸、上下文长度、并发三者互相挤压；
2. 小参数模型的工具调用可靠性明显低于旗舰模型，Agent 循环里 JSON 一出错就全盘崩溃；
3. 云端模型快在首 token，本地模型慢在每一步，长循环会把延迟放大数倍。

## 做法

**第一步：按硬件定模型档位。** 16GB 内存或 8GB 显存的机器，现实选择是 7B–14B 的 GGUF Q4_K_M 量化（性价比拐点）；24GB 显存可以摸到 32B。别信模型卡片上的"最大上下文"，长上下文的 KV cache 会吃掉大量显存，Agent 场景先把上下文压到 8K–16K 实测。

**第二步：选引擎，暴露 OpenAI 兼容端点。** Ollama 上手最快，llama.cpp 更可控，vLLM 适合有独显且要并发的场景。关键是它们都能提供 OpenAI 兼容 API，Agent 框架和 MCP 客户端只需改一个 base_url：

```bash
OLLAMA_HOST=0.0.0.0 OLLAMA_CONTEXT_LENGTH=16384 ollama serve
```

**第三步：把结构化输出钉死。** 小模型自由发挥 JSON 是事故主因。Ollama 用 `format` 传 JSON Schema，llama.cpp 用 GBNF 语法约束，vLLM 用 guided decoding。强制约束后，7B 模型的工具调用会从"看运气"变成"基本可用"。

**第四步：大模型规划，小模型执行。** 这是本地部署最重要的架构决策。让云端/旗舰模型只做任务拆解和复杂推理，把高频、模式化的调用（文件整理、定时任务、格式转换）路由给本地模型。Agent 循环里约八成调用是机械性的，这部分本地化收益最大。

**第五步：Embedding 单独跑。** RAG 和记忆检索用独立的小 embedding 模型（如 bge-m3），几百 MB 就够，别挤占主力模型的显存。

## 踩坑点

- **Ollama 默认上下文很短**：Agent 塞进一堆工具描述直接被截断，表现为"模型忘了系统提示"，先查 num_ctx；
- **量化太狠破坏格式输出**：Q2 档位即使加了 schema 约束也会吐残缺 JSON，宁可换小一档模型也别用 Q2；
- **并发默认为 1**：多个 Agent 同时请求会排队，设 `OLLAMA_NUM_PARALLEL` 时注意显存会近似翻倍；
- **模型冷启动**：默认 keep_alive 下模型几分钟就卸载，定时任务每次多等十几秒，高频任务设为 -1；
- **长循环复读**：温度调到 0.6–0.7 并加 repeat_penalty，否则小模型会陷入重复的工具调用循环。

## 可复用建议

1. 先用 `ollama ps` / `nvidia-smi` 建立显存基线，再逐步加上下文和并发，一次只动一个变量；
2. 在 Agent 框架里写一个"模型路由"配置：规划走强模型、执行走本地模型，云端 token 轻松省七成以上；
3. 把"schema 校验 + 失败重试"做成统一中间件，不要散落在每个工具里各写一遍；
4. 评测别跑通用 benchmark，直接用自己 Agent 的真实任务跑 20 条，看成功率和平均轮次，比 MMLU 有用得多。

## 总结

本地部署不是替代云端，而是给 Agent 体系加一层低成本、低延迟、数据不出门的执行层。家用硬件的现实上限是 7B–32B 量化模型，配合结构化输出约束和大小模型分工，足以承接大部分机械性工具调用。先小规模跑通一条链路，再把高频任务逐步迁下来——工程上稳妥，钱包也诚实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/e36bcf16535193ea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f5d23b943e9ff655.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/467d098abe503cb6.png)

