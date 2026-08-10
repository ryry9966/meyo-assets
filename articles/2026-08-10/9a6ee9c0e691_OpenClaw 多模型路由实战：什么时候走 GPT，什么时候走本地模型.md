---
title: OpenClaw 多模型路由实战：什么时候走 GPT，什么时候走本地模型
feedId: 32402
source: 综合讨论
publishedAt: 2026-08-10
---

在 OpenClaw 里把自动化流程拆成 Agent 和插件后，模型选择就成了一个绕不开的成本和延迟问题。全用 GPT-4o 一个 task 跑下来十几秒，token 消耗不少；全用本地模型，复杂推理又经常跑偏。本文聊聊 OpenClaw 下按任务特征做“多模型路由”的思路和落地方式，不绑 SaaS，完全跑在你自己环境里。

## 背景：单一模型为什么不够

OpenClaw 的典型使用场景类似这样：

- 用户给一个自然语言指令；
- OpenClaw 理解意图，拆成几个工具调用（浏览器、文件系统、API 等）；
- 执行一圈后再做总结或下一步决策。

这里至少包含三类不同难度的推理：

1. **意图理解与函数选择**：需要较强的指令遵循能力，但 prompt 短，通常能用便宜模型。
2. **复杂多步推理**：需要理解中间结果、识别失败并重试，必须用强模型。
3. **格式化输出与总结**：对创造力要求低，但需严格遵循 schema，很多小模型也可胜任。

如果所有请求走同一个模型，要么贵，要么不稳定。多模型路由就是在这几类任务间动态切模型。

## 问题定义：不是“用不用本地模型”，而是“什么特征用什么模型”

实践中容易掉进一个误区：试图先判断用户问题难不难，再决定派给谁。这在工程里很难做准。OpenClaw 里更可复用的做法是**基于“运行时特征”做路由**，而不是基于内容主题。

可用的特征包括：

- 当前是第几轮工具调用（第一轮 vs 中间重试）
- prompt 长度（短 prompt vs 超长上下文）
- 输出要求（自由文本 vs JSON 工具调用 vs 固定模板填充）
- 是否需要多语言或安全审核

把特征抽象出来，路由规则就稳定了，不受话题变化影响。

## 做法：在 OpenClaw 中插一层轻量 Router

OpenClaw 本身的配置允许你定义多个 LLM provider，我们可以在调用前插一个决策函数。下面是一个最小可行的实现思路，基于 OpenClaw 的插件体系。

### 步骤 1：注册多个模型

在 `config.yaml` 中定义两个后端，比如：

```yaml
llm_providers:
  - id: local-llama
    type: openai_compatible
    base_url: http://localhost:8080/v1
    api_key: not-needed
    default_model: llama3.1-8b-instruct
  - id: gpt4o
    type: openai
    api_key: ${OPENAI_API_KEY}
    default_model: gpt-4o
```

### 步骤 2：写一个 Router 插件

OpenClaw 支持在执行链中插入自定义插件。我们在 `pre_llm_call` 阶段做路由。伪代码：

```python
def route_model(task_context):
    # 特征提取
    turn = task_context.get("turn", 0)
    prompt_len = len(task_context.get("prompt", ""))
    requires_json = task_context.get("output_format") == "json"
    
    # 路由逻辑
    if turn == 0 and prompt_len < 2000 and not requires_json:
        return "local-llama"   # 简单意图解析
    if turn > 1 or prompt_len > 8000:
        return "gpt4o"         # 复杂重试或长上下文
    if requires_json:
        # 结构化输出如果本地模型稳定可用，也可走本地
        return "local-llama"
    return "gpt4o"
```

### 步骤 3：配置 fallback

本地模型偶尔会因为 OOM 或推理卡住而超时。Router 里要加一层兜底：同一个 task 如果本地模型 5 秒内没返回，自动降级到 GPT-4o‑mini 或 GPT-4o。这样用户侧几乎无感。

## 踩坑点

1. **本地模型的工具调用格式不稳定**

   很多本地模型（如 Llama 3.1 8B）在强制输出 JSON 时偶尔会多输出一个解释文字，导致 OpenClaw 的函数解析器报错。解决方法是 prompt 里加严格的格式要求，并在插件层做 retry：如果解析失败，用 GPT-4o-mini 补一次。

2. **上下文长度变化导致路由抖动**

   有时候 prompt 长度恰好卡在阈值附近，会出现同一个对话第一轮走本地模型，第二轮因上下文积累超限切到 GPT，第三轮又回到本地。这种频繁切换会让模型“人格”不一致。对策是：一旦某次对话切到 GPT 后，后续请求默认全走 GPT，除非显式重置。

3. **本地模型首次推理的冷启动**

   如果你用 Ollama 且模型没加载到内存，首次调用可能 3～10 秒才返回。Router 的超时判断需要把这个考虑进去，否则会误降级。可以用一个预热脚本在启动 OpenClaw 时先把常用模型加载好。

## 可复用建议

- **按“工具调用轮次”作为首要路由特征**：第一轮大概率是简单意图解析，后面的重试和整合则容易涉及复杂推理。
- **对结构化输出单独建模**：如果你的本地模型在 JSON mode 下表现稳定（如经过 fine-tune），单独设一条路由规则，能省大量 token。
- **统计实际命中率再调规则**：OpenClaw 可以把每次调用的模型名、耗时、是否成功记到日志。跑一两天后，看哪类请求本地模型失败率高，再调整特征条件。
- **长期考虑小模型专门化**：如果你有高频的特定工具调用，可以拿 500～1000 条真实数据对本地模型做 LoRA，之后这条路径完全不依赖 GPT。

## 总结

多模型路由在 OpenClaw 里不是“高级优化”，而是工程上负责任的做法。核心不是预测问题复杂度，而是根据可观测的运行时特征（轮次、长度、输出格式）做稳定分发，并配上无感降级。这样做下来，GPT 调用的比例可以控制到 30～50%，整体延迟和成本都会舒服很多，而用户体验几乎不降级。

---

