---
title: OpenClaw 的多模型路由：什么时候用 GPT 什么时候用本地模型
feedId: 31295
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在 OpenClaw 这类 Agent 框架中，模型不再是一个固定的云端 API 端点，而是一组可以按需切换的“大脑”。一边是 GPT-4、Claude 3.5 等闭源大模型，能力强但成本高、延迟不确定；另一边是本地部署的 Llama 3、Mistral、Qwen 等开源模型，免费、低延迟、数据不出域，但在复杂推理和工具调用上仍有差距。多模型路由就是为了在这两者之间找到工程上的最优解。

## 问题：什么时候该用哪个模型？

并不是所有请求都需要 GPT-4 级别的推理。例如：

- 意图分类、简单的文本摘要、情感分析，本地 7B/13B 模型完全够用。
- 包含敏感内部数据的代码审查或客户信息处理，必须走本地模型。
- 需要长上下文、多步骤推理、高质量代码生成、或依赖复杂 function calling 的任务，仍然要用 GPT-4 或同级的云端模型。
- 预算紧张或并发高峰时，需要自动降级到本地模型，保证服务不中断。

盲目把所有请求都扔给 GPT-4 会导致成本失控，一律用本地模型又可能让复杂任务失败。需要一套可复用的路由策略，这正是 OpenClaw 架构所支持的。

## 做法：在 OpenClaw 中实现模型路由器

OpenClaw 的设计允许为 Agent 配置多个模型后端，并通过自定义路由逻辑进行选择。以下是一个工程化的实现路径。

### 1. 定义决策因子

提取任务的关键属性，作为路由的输入：

- **任务类型**：如分类、生成、翻译、代码、工具调用。
- **复杂程度**：可由 prompt 长度、预计推理步骤数估算。
- **数据敏感度**：是否包含 PII、内部代码、凭证等。
- **工具需求**：是否需要 function calling、MCP 工具调用。
- **实时性要求**：是否要求低延迟（<500ms）或允许秒级延迟。
- **成本约束**：当前会话或日消耗是否已接近限额。

### 2. 构建路由逻辑

以一个典型 OpenClaw 配置为例，使用 `ModelRouter` 组件：

```python
# 伪代码，仅示意
class ModelRouter:
    def __init__(self, config):
        self.cloud_model = config['cloud_model']  # GPT-4
        self.local_model = config['local_model']  # llamafile / vLLM
        self.budget_cap = config['budget_cap']
        self.force_local_keywords = ['身份证', '内部代码']
    
    def route(self, task):
        # 规则1: 敏感数据 → 本地模型
        if any(kw in task.prompt for kw in self.force_local_keywords):
            return self.local_model
        
        # 规则2: 必须使用 function calling → 云端模型（除非本地模型支持）
        if task.needs_tool_calling and not self.local_model.supports_fc:
            return self.cloud_model
        
        # 规则3: 高复杂度 (token > 2000 或包含复杂指令) → 云端
        if len(task.prompt) > 2000 or "multi-step reasoning" in task.tags:
            if self.budget_remaining > 0:
                return self.cloud_model
            else:
                return self.local_model  # 预算耗尽，降级
        
        # 规则4: 简单任务 → 本地
        return self.local_model
```

在 OpenClaw 的 Agent 初始化时，将路由器注入：

```yaml
agents:
  worker:
    model_router: my_router
    backends:
      gpt4:
        type: openai
        model: gpt-4-turbo
      local_llama:
        type: openai_compatible
        endpoint: http://localhost:8080/v1
        model: llama3-8b
```

每条请求通过路由器动态选择 backend，OpenClaw 处理上下文转换和流式响应适配。

### 3. 用轻量分类器做前置筛选

为了避免每次都走大模型做路由判断，可以先用一个极低成本的方法（如关键词匹配、或一个量化的 BERT 分类器）决定任务类型，再结合规则路由。例如：

- 将用户输入送入本地 fastText 模型，得到意图标签。
- 根据标签 + 敏感词扫描 + token 计数，直接映射到模型选择。
- 只有标签置信度低时，才用一个小型本地 LLM 做二次判断。

## 踩坑点

1. **本地模型的工具调用支持不完善**  
   多数 7B 模型原生不支持 function calling，需要微调或借助 LangChain 等框架模拟。如果强制使用，容易出现 JSON 格式错乱，导致 Agent 解析失败。建议在路由时明确标注模型的能力矩阵。

2. **冷启动延迟**  
   本地模型若按需加载，首次请求可能耗时 15-30 秒（尤其 GGUF 文件挂在 HDD 上）。解决方式是采用常驻服务（如 vLLM、llama.cpp server），或使用 warm-up 探针保持活跃。

3. **上下文格式不一致**  
   云端和本地模型的 prompt 模板、角色定义可能不同。在 OpenClaw 中应使用统一的 `chat_template`，否则模型输出会出现角色错位。可通过预定义 adapter 处理。

4. **路由规则冲突**  
   当一条任务同时命中“敏感”和“需要工具调用”时，本应走本地但因为工具限制又必须走云端，产生矛盾。必须定义优先顺序：数据安全 > 功能完整性 > 成本。

5. **监控缺失**  
   初期路由策略往往靠拍脑门设定，缺少实际数据支撑。一定要在路由决策点打日志，记录模型选择、原因、token 消耗，方便后续优化。

## 可复用建议

- **策略配置化**：将路由规则写成 YAML/JSON，支持热更新，无需重启服务。
- **缓存常见请求**：对于确定性强的任务（如模板化回复），直接用 Redis 缓存结果，避免重复调用。
- **渐进式降级**：设定成本报警线，当日预算消耗 80% 时自动将部分低优先级任务降级到本地模型。
- **A/B 测试路由效果**：用 10% 流量测试本地模型替代云端模型，对比任务成功率与用户反馈，持续调整阈值。
- **能力矩阵表**：维护一张模型能力表（上下文长度、函数调用、代码能力、多语言质量），路由时直接查表，保障可维护性。

## 总结

OpenClaw 的多模型路由不是一个“选择 GPT 还是本地”的二元问题，而是一套需要持续调优的决策系统。通过合理抽象任务属性、制定优先级规则、结合轻量分类器和能力矩阵，可以在成本、隐私和功能之间找到一个工程上可持续的平衡点。务实一点的做法是：先让所有请求走云端，然后用一周的日志分析请求分布，逐步把低风险的任务切给本地模型，最终收敛到 80% 本地 + 20% 云端的组合。这样的架构既保持了系统的健壮性，也让成本可观测、可控制。

---

