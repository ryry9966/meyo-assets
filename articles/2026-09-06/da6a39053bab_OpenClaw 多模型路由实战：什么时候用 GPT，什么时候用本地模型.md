---
title: OpenClaw 多模型路由实战：什么时候用 GPT，什么时候用本地模型
feedId: 36259
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

在 OpenClaw 的 Agent 链路里，模型调用是成本和延迟的大头。我们的流水线中，既有需要多步规划、跨文件改代码的重活，也有大量“把这段日志抽成 JSON”的轻活。早期全部走 GPT，一个月账单里七八成花在了根本不需要顶级智力的事情上；反过来全切本地小模型，复杂任务失败率又明显上升。多模型路由解决的就是这个错配问题。

## 问题：按什么规则分流

OpenClaw 的 router 层支持配置模型池和回退链（fallback chain），但默认策略太粗。核心是三个问题：

1. 哪些任务对模型能力不敏感，本地模型够用？
2. 哪些任务必须上云端强模型？
3. 回退何时触发，怎么避免回退把省钱策略打穿？

## 做法

**第一步：给任务打标。** 在 task 元数据里加 `complexity` 和 `sensitivity` 两个字段：前者由调度器按工具调用深度预估，后者标记是否涉及内部文档、日志等不能出网的数据。

**第二步：按类分流。** 我们验证过的分界线：

- 本地模型（Qwen / Llama 系，Ollama 或 vLLM 部署）：格式转换、实体抽取、摘要、简单分类、tool-call 参数补全、高频低价值批处理。
- 云端 GPT：多步规划、跨文件代码修改、长上下文推理、面向用户的生成兜底、低频高价值任务。

**第三步：配置回退链。**

```yaml
router:
  routes:
    - match: { complexity: low }
      primary: local/qwen2.5-14b
      fallback: [cloud/gpt-4o-mini]
    - match: { complexity: high }
      primary: cloud/gpt-4o
      fallback: []
    - match: { sensitivity: internal }
      primary: local/qwen2.5-14b
      fallback: []   # 敏感任务禁止出网
```

**第四步：看指标，不看感觉。** 盯三个数：升级率（本地→云端的回退比例）、单任务平均成本、任务成功率。升级率持续高于 15%，说明任务打标有问题，路由规则该改了。

## 踩坑点

- **本地模型上下文窗口小。** Agent 循环跑到十几轮，历史一塞进去就截断，表现为“模型突然变傻”，实际是上下文被砍了。对本地路由的任务要限制循环轮数或做历史压缩。
- **tool-call 格式质量参差。** 部分本地模型输出的 JSON 参数偶尔带注释或尾逗号，插件解析直接失败。我们在插件入口加了一层宽松解析加一次自动重试。
- **回退链写得太宽等于没省钱。** 某次升级率冲到 40%，排查发现是 complexity 预估系统性偏低，本地模型全被顶到了云端。
- **敏感任务挂云端 fallback 等于数据出网。** `sensitivity: internal` 的路由必须显式清空 fallback，依赖默认配置一定会漏。

## 可复用建议

- 路由规则放配置不放代码，改规则不需要重新发布。
- 建一个 50~100 条的回归集，每次调路由先跑一遍，别靠单例试。
- 模型版本钉死，本地模型升级单独走灰度。
- 每类任务记录“模型 × 成功率 × 成本”三元组——路由优化本质上是在这张表上做搜索。

## 总结

多模型路由不是“本地替代云端”，而是承认任务有档次。在 OpenClaw 里先把任务分类做扎实，再配路由和回退链，最后用升级率和成本指标闭环校准。建议从最不敏感的那批批处理任务切本地开始，通常一个月内就能在账单上看到变化，再逐步扩大范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9d48fe8607dde571.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/0c5d40263e6c7cf3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/e8338e6b78a888ee.png)

