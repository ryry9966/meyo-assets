---
title: OpenClaw 多模型路由实战：什么时候该用 GPT，什么时候用本地模型
feedId: 31663
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

OpenClaw 这类 Agent 框架的最大优势是"什么都能接"：OpenAI、Anthropic、本地 Ollama、任何兼容 OpenAI 协议的私有端点，都能配成 provider。但"能接"和"会用"是两码事。不少人的配置里只有一把锤子——要么全走 GPT，账单每月见涨；要么图省事全走本地模型，复杂任务输出质量明显掉档。

问题不是"哪个模型更好"，而是"什么任务该交给哪个模型"。这个判断应该有规则，而不是靠感觉。

## 问题拆解

拆开看，OpenClaw 里的任务可以按三个维度分类：

- **复杂度**：简单意图识别、工具调用 vs 多步推理、长文本写作。
- **隐私敏感度**：涉及本地文件、邮件、数据库内容的任务，不该把全文丢给云端。
- **成本/延迟预算**：高频低价值操作（状态查询、格式化输出）用便宜模型；低频高价值操作（方案生成、代码审查）用强模型。

## 做法：给路由写规则

我在 OpenClaw 配置里挂了两个 provider：本地 Ollama（Qwen2.5-7b-Instruct）和 GPT-4.1-mini（轻量档）。路由不靠外部网关，直接在 Agent 层用一个入口意图分类器实现。

大致链路：

1. 用户输入进入入口分类器（用本地小模型跑）。
2. 分类器输出标签：`tool_parse` / `quick_answer` / `complex_reasoning` / `private_context`。
3. 按标签分流：
   - `tool_parse`、`quick_answer` → 本地模型；
   - `complex_reasoning`、`code_review` → GPT 系列；
   - `private_context` → 强制本地，并标记敏感上下文，禁止外发。

配置上，就是给每条规则绑定 provider 和 model 字段：

```yaml
router:
  rules:
    - match: "intent:tool_parse|quick_answer"
      model: "ollama/qwen2.5:7b-instruct"
      max_tokens: 512
    - match: "intent:complex_reasoning"
      model: "openai/gpt-4.1-mini"
      fallback: "openai/gpt-4.1"
    - match: "intent:private_context"
      model: "ollama/qwen2.5:14b-instruct"
      no_cloud: true
```

分类器本身开销很小，高频场景用 `max_tokens: 64` 限制输出长度即可，别让它写解释。

## 踩坑记录

按损失程度排：

1. **本地模型的 function calling 格式不兼容**。Qwen2.5 的 tool call 和 OpenAI 格式有细微差异，OpenClaw 走 OpenAI 兼容端点时会收到 `invalid tool_calls`。解决：本地模型只负责分类、不直接发 tool call，参数解析交给 GPT 或后端逻辑。
2. **延迟不是越低越好**。M 系列 Mac 上跑 7B 模型单次推理确实快，但多轮对话下显存交换频繁，整条链路 P95 反而高于云端。路由规则里要加"连续调用次数"阈值，超了就切云端。
3. **自动路由别只看 token 数**。我一度用输入长度做阈值，结果长日志文件全被丢给 GPT，账单暴涨——而这些任务其实只需要 grep 式提取。
4. **切换模型导致上下文风格突变**。本地模型回答简短，GPT 回答详尽，会话中途切换用户会明显感知到。后来在 system prompt 里加了输出风格规范，两边统一"先结论后展开"。

## 可复用建议

- **先分类，再路由**。分类器用本地小模型，速度和隐私两头兼顾。
- **给每条规则配 fallback 链**：本地 → 轻量 GPT → 重模型，逐级兜底。
- **记录路由决策**。日志里输出 `route: local|gpt` 和耗时，跑一周再调阈值，不要拍脑袋定。
- **敏感数据用"标记"而不是"猜测"**。在 tool 层或 sidecar 上打标签，路由只看标签。

## 总结

多模型路由不是炫技，是对成本、延迟和隐私的显式约束。OpenClaw 给了你挂多个 provider 的自由，但"用哪个"的决策权得收回到规则里。先跑起来，再根据日志调，比一开始就搭复杂的智能路由网关靠谱得多。

实测下来，一个简单的三规则分流器，能把月度 API 成本压掉约 60%，同时复杂任务输出质量无明显下降。这个收益，值得花半天时间配置。

---

