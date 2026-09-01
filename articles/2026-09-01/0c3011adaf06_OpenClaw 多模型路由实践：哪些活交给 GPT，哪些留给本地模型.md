---
title: OpenClaw 多模型路由实践：哪些活交给 GPT，哪些留给本地模型
feedId: 35685
source: 综合讨论
publishedAt: 2026-09-01
---

# OpenClaw 多模型路由实践：哪些活交给 GPT，哪些留给本地模型

## 背景

Agent 跑起来之后，调用量远比想象中大：一次任务里 planner、工具参数生成、摘要、抽取、embedding 各来几轮。全走云端，账单和首 token 延迟都难看；全切本地，planner 和多步工具调用的质量立刻下滑。多模型路由解决的就是这个两头不讨好。

## 问题

常见的错误做法有两种：按流量随机分配，导致同一场会话风格漂移、上下文理解不一致；或者"本地优先、失败上云"，本地一卡，所有请求涌向云端，成本瞬间打穿。路由的正确粒度是**任务类型**，不是请求。

## 做法

1. **先分类，再路由。** 在 OpenClaw 的 trace 里给每次调用打 `task_type` 标签：planner、toolcall、summarize、extract、embedding。这是所有规则的地基，没有这一步，后面全是拍脑袋。
2. **定义模型池。** 云端放一个强模型；本地用 vLLM 或 Ollama 起一个 30B 级模型，两边注册进 model pool。
3. **写规则。** 原则：planner 和 toolcall 走云端——这两类错一次的返工成本远高于 token 差价；summarize、extract 这类高频低难度调用走本地；embedding 固定本地。规则带上下文长度条件，超本地窗口自动升级云端。
4. **配 fallback 和预算。** 本地超时或连续失败后切云端，但每会话设云端调用上限，防止故障期成本失控。
5. **看数据再调。** 跑一周，看每类任务的通过率和返工率，再移动边界。

```yaml
# 示意，字段名以你版本的 schema 为准
router:
  rules:
    - { match: { task_type: planner },   use: cloud }
    - { match: { task_type: toolcall },  use: cloud }
    - { match: { task_type: summarize, ctx_lt: 8k }, use: local }
    - { match: { task_type: extract },   use: local }
  fallback: [local, cloud]
  budget: { cloud_calls_per_session: 40 }
```

## 踩坑点

- **本地模型的工具调用参数不可靠。** 30B 级模型生成 MCP 工具的 JSON 参数时，字段缺失、多余引号是常态，插件 schema 校验一严整条链就断。要么在本地出口加严格校验加重试，要么把 toolcall 钉死在云端——我们选了后者，省心。
- **上下文静默截断。** 本地 8k、云端 128k，长文档摘要不检查长度就路由到本地，输出看似正常实则丢了一半信息。长度条件必须写进规则里。
- **本地并发排队引发 fallback 风暴。** 单卡 30B 并发有限，多 agent 同跑时延迟从 2 秒涨到 20 秒，超时触发 fallback，云端成本反超"全云"方案。给本地池设并发上限，超出的直接走云端，不要排队。
- **会话风格漂移。** planner 和正文用不同模型，多轮下来语气会变。做会话级 sticky routing 能缓解。

## 可复用建议

- 路由表当代码管，进版本库，改动走 PR，回滚有据可查。
- 每次路由决策写进 trace，排障时能回答"这条请求为什么去了云端"。
- 先用两个模型跑通，再考虑第三个；模型每多一个，排障复杂度不止翻倍。
- 维护一份"质量敏感任务清单"（对外输出、涉及真实操作的调用），永久钉在云端，不参与路由。

## 总结

多模型路由不是省钱小技巧，而是把"哪些调用值得花好模型的钱"变成显式的工程决策。在 OpenClaw 里做对三件事就够：按 task_type 分流、带长度条件的规则、有预算护栏的 fallback。剩下的交给 trace 数据去迭代，比任何预设的"最优配比"都可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/7e803644dd62f169.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/fee46ade39478b68.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/6953c61646649c72.png)

