---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好，而不是每次重新自我介绍
feedId: 34611
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

很多 OpenClaw、Agent 和 MCP 用户在接自动化流程时都会遇到同一类问题：每次新会话都要重新告诉助手“输出用中文”“不要写 docstring”“默认走 test 分支”。等它终于“学会了”，会话一结束又清零。

于是有人开始把聊天记录塞进上下文，或接一个 memory 插件无脑写入。结果不是 token 爆炸，就是检索出一堆无关片段。

## 问题

记忆不是存档。Agent 真正需要的“记住偏好”，是让稳定偏好低成本进入决策上下文，让临时信息不污染长期记忆。

常见错误有三个：把所有对话当记忆；把偏好和事实混在一起；写入后无法纠错。

## 做法 / 步骤

### 1. 先分层，不要只做一个“记忆文件”

我通常把 Agent 记忆分成四类：

- **会话工作记忆**：当前任务中的变量、暂存结果，生命周期为一个 session；
- **用户偏好层**：稳定、跨任务，比如语言、回答风格、工具链偏好；
- **事实/项目记忆**：路径、环境名、部署位置、账号策略等；
- **流程/经验记忆**：可检索的排障记录、操作步骤摘要。

前两类适合直接注入 system prompt 或临时上下文；后两类适合按 key 存储或做检索。分层后，一个关键问题就能解决：不是所有记忆都值得全量加载。

### 2. 用结构化 schema 写偏好，而不是写散文

最小实现可以是一个 JSON 文件：

```json
{
  "preferences": {
    "language": "zh-CN",
    "output_style": "concise",
    "default_test_branch": "test/current"
  },
  "facts": {
    "project_path": "/data/openclaw",
    "deploy_env": "staging"
  }
}
```

每条记录带 metadata：`id`、`type`、`created_at`、`updated_at`、`confidence`、`source`。这个 schema 不复杂，但能避免“用户随口说的一句话”被当成永久偏好。

### 3. 写入要有触发条件，不要无脑落库

我推荐的规则是：当用户明确纠正、确认或使用 `/remember` 命令时才写入。

“以后都用中文”是偏好；“这次用中文”不是。

任务完成后可以让 Agent 提取一次差异，但必须标记为低置信度，待用户确认。这样能显著减少记忆污染。

### 4. 检索时把偏好和事实分开

偏好层通常很小，直接全量注入；事实/流程记忆才走检索。

检索时加过滤：`type`、`project`、时间范围。不要把“我喜欢简短回答”和“测试库地址是 xxx”放在同一向量空间里打分。

在 OpenClaw 里可以给 MCP memory server 暴露三个工具：`read_memory`、`write_memory`、`search_memory`，让 Agent 按类型调用。

### 5. 更新和纠错比写入更重要

同一个 key 更新时保留时间戳和来源；冲突时，用户最近的确认优先。

必须提供 `/forget` 或 `delete_memory` 工具。否则 Agent 记住一个错误路径后，用户只能去删文件或改配置，这在自动化里很危险。

## 踩坑点

- 把全量聊天记录当长期记忆：上下文膨胀，检索结果经常答非所问；
- 写入太频繁：用户在调试过程中的临时选择被固化为偏好；
- 没有分层：一条“今天先跑测试”被长期记住；
- 不做可编辑/可擦除：错误记忆无法纠正，用户逐渐不信任 Agent；
- 隐私泄露：记忆文件里可能有 token、内部路径，不要默认同步到云端或贴进日志；
- 偏好注入过度：把所有记忆都放进 system prompt，导致指令遵循下降。

## 可复用建议

- 先做一个 Markdown/JSON 文件 + 简单读写工具，跑通“写入—注入—纠正”闭环，再考虑向量库；
- 写前先读：写记忆前先查已有 key，避免重复或冲突；
- 给记忆文件做版本管理，定期 review，像管理配置一样管理偏好；
- 在 OpenClaw 插件里，session start 加载记忆，session stop 异步写入提取到的偏好；写入动作最好可审计。

## 总结

Agent 记忆系统的核心不是“记得多”，而是“该记住的稳定、可检索、可纠正、可解释”。

从分层和结构化 schema 开始，用文件系统做基线，控制写入触发和检索范围，就能解决大多数偏好记忆问题。等量大了，再上向量检索或专门 memory 服务也不迟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e1a35e52de34fa39.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/baecdbea8a18c747.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2a4ea127104d9c7e.png)

