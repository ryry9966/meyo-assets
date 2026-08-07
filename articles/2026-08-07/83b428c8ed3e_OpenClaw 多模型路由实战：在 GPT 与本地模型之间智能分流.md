---
title: OpenClaw 多模型路由实战：在 GPT 与本地模型之间智能分流
feedId: 32000
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在用 OpenClaw 搭建 AI Agent 时，很多团队都会面临一个典型选择：任务到底该发给 GPT-4 这类远端大模型，还是跑在 Ollama 上的本地模型？  
远端模型能力强、支持工具调用和长上下文，但调用有成本、延迟高、数据出境有合规隐患；本地模型免费、响应快且数据不出机，但推理能力弱、缺乏稳定的函数调用能力，上下文窗口也有限。

OpenClaw 作为多模型编排框架，天然提供了模型路由的能力。我们可以按照任务属性、成本约束和功能需求，在框架层定义“什么请求走 GPT、什么请求走本地模型”，而不用在业务代码里硬编码判断逻辑。这篇文章就把我在项目中实际落地的路由器配置、踩过的坑和可复用的经验写出来，供社区参考。

## 问题

我们希望实现这样一个路由行为：  
- 用户说“今天天气怎么样”这类简短、无工具调用的日常对话 → 交给本地 Llama 模型，省成本、低延迟。  
- 遇到“帮我从文档里提取关键信息并生成表格”这类复杂任务，或显式声明了工具使用（如搜索、读写文件）→ 自动切到 GPT-4，确保执行质量。  
- 本地模型出错或超时时，能够自动降级到远端模型，避免任务中断。

在 OpenClaw 的配置体系中，改如何低代码地实现？

## 实践步骤

### 1. 启用 OpenClaw 模型路由插件
OpenClaw 提供了 `@openclaw/router` 插件（需要确保版本 ≥ 0.8.0）。安装后，在 `agent.yaml` 里引入：

```yaml
plugins:
  - name: router
    type: @openclaw/router
```

### 2. 定义模型后端
在 `models.yaml` 中分别声明远端和本地模型：

```yaml
backends:
  gpt4:
    provider: openai
    model: gpt-4-turbo
    api_key: ${OPENAI_API_KEY}
    max_retries: 2
  local_mistral:
    provider: ollama
    model: mistral:7b-instruct-v0.3
    base_url: http://localhost:11434
    context_window: 8192
```

### 3. 编写复合路由策略
路由策略的核心是规则链。在 `router` 插件配置里，我们采用 `composite` 策略，依次匹配条件：

```yaml
router:
  strategy: composite
  rules:
    - name: tool_required
      condition: "has_tools == true"
      model: gpt4
      priority: 1
    - name: long_context
      condition: "estimated_tokens > 800"
      model: gpt4
      priority: 2
    - name: simple_reply
      condition: "estimated_tokens <= 800 and has_tools == false"
      model: local_mistral
      priority: 3
  fallback: gpt4
  fallback_on_error: true
```

- `has_tools`：OpenClaw 会自动检测当前意图是否声明了工具调用（比如 MCP 工具或插件里定义的操作）。
- `estimated_tokens`：框架会用本地分词器预估输入 token 数，虽然不完全精确，但足以区分类似“写一封邮件”和“总结一篇长论文”的任务。
- 优先级从高到低匹配，一旦命中即停止。
- `fallback` 指定所有规则未命中或命中模型不可用时的兜底模型；`fallback_on_error: true` 可捕获本地模型超时、503 等异常并自动重试到 GPT。

### 4. 验证路由效果
启动 Agent 后，用 OpenClaw CLI 发送不同请求并观察日志：

```bash
openclaw chat -m "今天忙吗？"            # 应走 local_mistral
openclaw chat -m "搜索最新的 Rust 异步运行时" --tool search  # 应走 gpt4
```

日志中会打印 `[router] matched rule: simple_reply -> local_mistral` 等信息。

## 踩坑记录

**本地模型对工具调用的支持不成熟**  
社区里的本地模型（如 Mistral 7B）虽可通过 prompt 模拟函数调用，但输出格式不稳，容易导致 OpenClaw 解析失败。如果路由规则没有显式优先 `has_tools`，容易让工具请求误入本地模型导致任务中断。我们的做法是让所有涉及工具的请求无条件走 GPT，并在简单闲聊规则中明确 `has_tools == false`。

**分词预估不准造成误判**  
用本地分词器估算 token 数时，如果对话历史里包含大量代码块或特殊字符，预估值可能偏低，导致本应走 GPT 的长上下文任务被发给本地模型，进而本地模型截断输出。建议在 `estimated_tokens` 阈值上留 20% 缓冲，或者为代码相关对话引入单独的 intent 标签。

**远端模型成本飙升**  
当 fallback 频繁触发时，调用量会在短时间内冲到 GPT-4 上，尤其并发较高的时候。必须为 OpenAI 的后端配置速率限制和预算告警。我们在 OpenClaw 的 `backends.gpt4` 里增加了：

```yaml
rate_limit:
  max_requests_per_minute: 30
  on_limit: queue
billing_alert_threshold: 5.0  # 单日成本超过 5 美元时告警
```

**路由器自身开销**  
复合策略的规则链匹配几乎无开销，但如果日后想接入语义路由（例如用小模型做意图分类），需要把这部分推理时间算进首 token 延迟。一开始用简单的规则组合完全够用，别过早优化。

## 可复用建议

1. **采用“本地优先 + 智能升级”的渐进式路由**  
   日常对话全部走本地，复杂任务、工具调用和本地模型错误时再上升至 GPT，最大化节省成本。

2. **记录路由决策元数据**  
   在 OpenClaw 日志中保留每次路由选择的规则名、候选模型及耗时，便于后续分析成本分布和误判案例。可以用 `trace_id` 串联整个调用链。

3. **为高频静态响应设置缓存**  
   对于一些不依赖上下文的固定话术（如问候、命令介绍），可以在路由器前三层缓存直接返回，避免浪费模型调用。OpenClaw 支持前置缓存适配器，配置简单。

4. **定期回归规则有效性**  
   每两周随机抽样 200 条路由日志，人工确认是否存在规则遗漏。随着工具增多和模型升级，过去“长上下文一定走 GPT”的规则可能变得不再必要。

## 总结

OpenClaw 的模型路由不是一个做成后就可以忘掉的配置，它更像是 Agent 的“调度大脑”，需要根据业务场景、模型成本和能力边界持续调优。从工程角度看，用最少的规则覆盖 90% 的流量，再结合降级、监控和缓存，就能在保持体验的同时将远端模型调用量压缩到最低。希望这篇实践记录能给正在做模型编排的朋友一些实在的参考。

---

