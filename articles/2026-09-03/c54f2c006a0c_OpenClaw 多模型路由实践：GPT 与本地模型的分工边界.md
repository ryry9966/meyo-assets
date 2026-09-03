---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界
feedId: 35930
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的 agent 任务差异很大：一条流水线里，既有需要多轮工具编排、代码级推理的主控任务，也有大量"读 JSON、抽字段、写摘要"的机械调用。全部走 GPT，账单和延迟都难看；全部压给本地小模型，规划环节就会反复返工。多模型路由的价值，就是把这两类调用拆开，而不是笼统地"选一个更强的模型"。

## 问题

难点不在"能不能接多个模型"，而在路由决策放在哪一层：

- 运行时"先小模型、不行再升级"的试探式路由，失败往往是静默的——agent 拿到似是而非的结果照样往下走，排障成本极高；
- 按对话轮次切换模型，上下文记忆格式不一致，长会话容易崩；
- 本地模型的上下文窗口、结构化输出支持参差不齐，路由前不校验等于埋雷。

## 做法

**1. 按任务类型分桶，不要按"难度"猜。**

- 规划、工具编排、代码修改 → GPT（前沿云端模型）；
- 摘要、字段抽取、分类、格式转换、embedding → 本地模型（Ollama/vLLM 起的 Qwen、Llama 系列）；
- 涉及内部日志、客户数据、密钥周边的调用 → 一律本地。

**2. 路由绑定在任务/子代理粒度。** OpenClaw 里每个 agent 可以单独指定 model profile，MCP 工具封装时也可以在工具层指定模型。这样路由是显式、可审计的，而不是运行时猜的。配置示意（字段名以你手头版本的文档为准）：

```yaml
models:
  planner: { provider: openai, model: gpt-4.1 }
  worker:  { provider: ollama, model: qwen2.5:14b-instruct }

routing:
  default: worker
  overrides:
    plan: planner
    tool_orchestrate: planner
    summarize: worker
    extract_fields: worker

escalation:
  on: [schema_fail, timeout]
  retries: 1
  then: planner
```

**3. 建一个回归集。** 从真实 trace 里抽 30–50 条代表性样本，路由改动前后各跑一遍，diff 输出质量。没有这个，任何路由策略都是玄学。

## 踩坑点

- **静默降级最贵**：worker 输出不合法 JSON → schema 校验失败 → 重试仍失败 → 升级到 GPT，一来一回比直接用 GPT 还慢。升级链路必须有次数上限，并打点记录每一次路由决策。
- **上下文窗口**：本地模型 32k 窗口，长对话 transcript 塞不进去。路由前先算 token 预算，超限直接走云端，别指望截断之后质量还行。
- **单轮更快 ≠ 任务更快**：小模型生成快，但多绕两轮工具调用，端到端时延反而输。看任务总耗时，不要看单 token 速度。
- **别在会话中途换模型**：路由切分点放在任务边界，不要放在对话轮次上，否则记忆格式对不上。
- **统一结构化输出校验**：本地模型的 function calling / JSON 输出稳定性差异很大，一律走 JSON Schema 校验 + 有限重试，别信"它应该会输出 JSON"。

## 可复用建议

- 一句话原则：**强模型管规划和写代码，便宜模型管"读进读出"的体力活。**
- 路由要显式：绑定任务类型，日志里记录每次调用的 model 和路由依据，事后才查得动。
- 成本按任务类型统计，不要按模型统计——你才能真正看到钱花在哪。
- 升级阶梯保留但收紧：本地 → 重试 1 次 → GPT，够了。层级越多，排障越难。
- 本地选型只看三件事：结构化输出稳定性、上下文窗口、你硬件上的实际吞吐。参数量排名不重要。

## 总结

多模型路由不是省钱技巧，而是把"哪些调用需要智力、哪些只需要服从"分清楚。显式路由 + schema 校验 + 回归集，三件套齐了之后，GPT 和本地模型各干各的，成本和时延都会回到一个可解释、可优化的水平。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/582db46cb39d6b05.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/217121bb0bd6edec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/57cf7c108f02d5a8.png)

