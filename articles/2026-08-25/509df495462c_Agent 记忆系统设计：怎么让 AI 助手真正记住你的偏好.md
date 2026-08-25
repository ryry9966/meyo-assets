---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 34737
source: 综合讨论
publishedAt: 2026-08-25
---

# Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好

很多跑 OpenClaw 自动化的人都有这种体验：Agent 工具链路很顺，但每次新开会话就像失忆。你上次说过“部署目标先 staging”“日志输出 JSON”“不用 Docker 用 Podman”，下次它又按默认来。问题通常不在模型能力，而在于记忆没有被当成一个工程组件设计。

## 背景：有状态，但不是把所有东西都记住

Agent 运行时会碰到四类信息：

- 会话上下文：本轮任务正在发生什么；
- 长期偏好：用户长期、跨任务的稳定选择；
- 任务临时状态：某个工作流跑到哪一步；
- 工具结果缓存：避免重复调用。

很多人把长期偏好塞进 system prompt 写死，或者更糟，靠用户每次手动交代。真正可用的是：把偏好从 prompt 抽出来，做成结构化数据。

## 做法：先建一个小型偏好存储

最小结构建议 JSON 或 YAML 文件，每条偏好包含 key、value、scope、source、updated_at：

```json
{
  "preferences": [
    {
      "key": "preferred_shell",
      "value": "fish",
      "scope": "shell",
      "source": "user_explicit",
      "updated_at": "2025-04-01T10:00:00Z",
      "confidence": 0.9
    }
  ]
}
```

不要只存自然语言句子。key/value/scope 让检索和冲突处理变得可预测。

其次，在 Agent 启动或任务前，注入“记忆摘要”，而不是全量。OpenClaw 里可以在编排层读取偏好文件，按当前任务类型过滤：代码任务注入 shell、提交规范、包管理器；日志任务注入日志格式、时区、输出目标。

第三，用 MCP 暴露记忆。可以写一个小 MCP server，把上面的文件作为 resource 暴露，例如 `memory://preferences`。这样主 Agent、子 Agent、插件都读同一份，不会出现 A 插件记得住、B 插件又忘了。

第四，记忆更新要走工具调用。用户说“以后都用 uv 代替 pip”时，Agent 不应只在回复里说“好的”，而应调用 `update_preference` 工具写回文件。写回时带上时间戳、来源，最好保留旧值。

最后是冲突处理。不要新偏好永远覆盖旧偏好。可以按 scope 区分：同一个 key 且同 scope 才冲突；冲突时按来源置信度排序——用户明确指示 > 任务上下文推断 > 历史默认。低置信度或跨项目冲突时，让 Agent 反问一次：“这是全局默认，还是只针对当前项目？”

## 踩坑点

最常见的错误是把所有对话摘要丢进长期记忆。Token 会爆，检索也很慢。记忆应保留决策、约束、偏好，不需要保留完整对话。

另一个坑是“想起来才更新”。如果偏好只存在于某次对话 summary 里，后面检索靠运气。结构化文件和显式更新比模型自觉靠谱。

时间衰减也必须考虑。用户三年前说“用 Python 3.8”，现在项目已经 3.12，旧偏好会造成干扰。建议给每条记忆加 `expires_at` 或定期复核。

敏感信息不要进偏好文件。token、密码、内网地址不要随 prompt 到处注入。偏好文件里可以只存引用名，真实值放 secret 管理。

## 可复用建议

- 最小字段：key、value、scope、source、updated_at；再加 confidence、expires_at 足够。
- 用 MCP resource 暴露，避免环境变量或硬编码；可审计、可共享。
- 更新走工具调用，带时间戳和来源。
- 每次检索只取 top N 相关条目，避免上下文污染。
- 偏好文件纳入版本控制，方便回溯改动。
- 增加“全局/项目/会话”三级 scope，减少误覆盖。

## 总结

让 Agent 记住偏好，不靠更大的 prompt，而靠结构化存储、按需注入、显式更新和版本化。工程上先做一个小 JSON 偏好文件，用 MCP 暴露，更新走工具，注入做摘要，字段带时间戳和来源。这样它才会在正确的时间拿到正确的约束，而不是每次重新交代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c8c26f9615b0094b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ab22a48c34c9f753.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/60696cbeaa615920.png)

