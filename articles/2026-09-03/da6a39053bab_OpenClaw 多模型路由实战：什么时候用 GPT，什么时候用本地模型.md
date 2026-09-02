---
title: OpenClaw 多模型路由实战：什么时候用 GPT，什么时候用本地模型
feedId: 35885
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 默认是“一个主力模型走天下”，能跑，但账单和隐私问题会慢慢暴露。我把网关上两个月的 session 日志过了一遍，抽样统计后发现：约六成请求是摘要、翻译、格式化这类轻量任务，用 14B 本地模型完全够用；真正需要强推理和复杂 tool-calling 的不到两成。钱和隐私，都花在了不该花的地方。

## 问题

三个痛点：

1. **成本**：重复性任务持续烧 token，大部分是低价值输出；
2. **隐私**：本地文件、聊天记录默认送云端；
3. **能力错配**：反过来，本地小模型跑多轮工具调用经常翻车。

核心问题是路由策略按什么切——按价格、按延迟，还是按任务类型。

## 做法

**第一步：先分类，别急着配模型。** 从日志抽 200 条请求打标签，分三档：轻量转换、中等推理（写脚本、解释报错）、重度 agent（多轮工具调用、MCP 长输出处理）。

**第二步：把本地模型挂进来。** Ollama 起一个 Qwen2.5-14B（按显存选型号），作为 OpenAI 兼容 provider 写进 `openclaw.json`：

```json
{
  "models": { "providers": { "ollama": {
    "baseUrl": "http://127.0.0.1:11434/v1",
    "api": "openai-completions" } } },
  "agents": { "defaults": { "model": {
    "primary": "anthropic/claude-sonnet-4-5",
    "fallbacks": ["ollama/qwen2.5:14b"] } } }
}
```

**第三步：选路由方式。** 简单场景用多 agent 拆分——不同 agent 绑不同模型，入口插件按意图分发；要更细的粒度就用 hooks 写一个前置分类器，在请求发出前改写 model 字段。注意 fallback 链方向：轻量任务本地优先、云端兜底；隐私敏感 session 要 fail-fast，本地失败就报错，不能悄悄回落到云端。

**第四步：建评测和观测。** 一个 30 条左右的黄金测试集，每次改路由跑一遍，记录正确率和延迟；线上按 session 记录实际使用的 model 字段，每周看一次占比。

## 踩坑点

- **tool-calling 兼容性是第一大坑**：本地模型 function call 格式差异会导致 OpenClaw 解析失败，表现为“不调工具直接回答”。选 tool-calling 成熟度高的模型，Qwen2.5 系列实测比多数小众模型稳。
- **MCP 工具返回的大 JSON 容易撑爆小上下文**。给绑定本地模型的 agent 限制工具返回长度，或先做一轮裁剪。
- **embedding 也是路由的一部分**：RAG 检索用本地 embedding 就够，别顺手走云端。
- **显存与并发**：14B + 长上下文在 8G 卡上很吃力，并发一上来延迟比云端还高，路由就失去意义了。
- 路由规则要能 diff、能回滚，改路由和改代码同等对待。

## 可复用建议

- 按**“复杂度 × 数据敏感度”**二维矩阵定路由，而不是按模型单价；
- 默认保留一个强模型，往“省”的方向优化，而不是把省的配置当默认；
- 每条 session 记录 model 字段，这是一切后续优化的数据基础；
- 路由分类器本身用本地小模型就够，分错了顶多多一次调用，不值得上大模型。

## 总结

多模型路由不是炫技，而是把“哪些请求值得付费推理”变成可度量的工程问题。两个月下来，我的云端 token 花费降了约六成，隐私任务全部留在本地，重度 agent 任务的质量没有可感知的下降。别从配置开始，从日志分类开始——一个下午就能出第一版路由规则。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/951cf74ff7006348.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/e96866d9c2b06d68.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/6bd3a6b1c359282a.png)

