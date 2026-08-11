---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 32701
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：为什么 Agent 的记忆会成为瓶颈

在 OpenClaw 这类 Agent 框架里，大家先把注意力放在工具调用、多步推理、MCP 协议集成上，等到实际跑长周期任务时，会很快撞上一个问题： **Agent 每次都像第一次见到你一样。**

比如你让 Agent 每天生成一份符合你审美的短报，第一次它会输出一堆废话排版；你手动指定“标题用中文、来源放末尾、不要 emoji 堆砌”，它当时记住了，明天再跑一次，又是初始风格。这不是模型能力问题，是记忆没被工程化。

记忆不是“ChatGPT 那样的长上下文”，而是 **结构化偏好、历史决策结果、人与 Agent 的交互修正信号的可持续存取**。这里不讨论 RAG 长文本那一套，而是聚焦在做自动化任务时，怎么让 Agent 真正记住你的偏好并能增量修正。

## 问题拆解：记忆在 Agent 流水线中的位置

一个典型的 OpenClaw 任务链如下：

触发条件 → 意图路由 → 工具选择 → 执行 → 结果生成

记忆需要在三个节点发挥作用：

1. **工具选择与参数填充** —— 比如你总是用某个 API 图床而不是另一家；
2. **执行策略偏好** —— 是否要先草稿后润色，是否禁止某些网站；
3. **输出风格与格式** —— 排版、语气、信息密度、是否附原始链接等。

目前大多数人尝试的做法是直接塞系统提示词，效果很差，因为：系统提示词是静态的，不能按任务语境增量修正，也无法跨会话持久化。而“写入文件让 Agent 读”又缺乏结构化查询能力。

## 工程化做法：三层记忆模型

我们实际落地的一套轻量方案，分为三层：

### 第一层：Profile（人设级偏好）

记载“总是如此”的元偏好，以结构化 YAML 形式存于本地：

```yaml
user_profile:
  language: zh-CN
  tone: concise
  output_format:
    news_digest:
      sections: [headline, summary, source]
      no_emoji: true
  default_tools:
    image_upload: local_api
    search: searxng
```

Agent 在每次任务开始时，通过 MCP 的 `read_resource` 能力加载这个文件，并注入到工作上下文最前面（不是系统提示，而是作为第一条 user 消息的上下文块）。这样做的好处是可以在运行中被 MCP 工具修改（例如用户说了“以后别用那个图床了”，Agent 可以通过脚本更新 profile.yaml）。

### 第二层：Task Memory（任务级记忆）

针对特定类型的重复任务（比如“每日短报”），维护一个 JSON 任务记忆文件：

```json
{
  "task": "daily_digest",
  "last_prompt_used": "...",
  "preferred_sources": ["HN", "arXiv", "国内快讯"],
  "disliked_sources": ["reddit"],
  "format_history": [
    {"date": "2025-01-20", "user_feedback": "source顺序放前面"}
  ]
}
```

每次执行任务前，Agent 读取对应 task memory，把上次使用的 prompt 与用户反馈纳入指令。关键点在于 **user_feedback 的增量叠加**：不是重写整个 prompt，而是以追加 diff 的形式：“把 source 放到正文前，不要放到末尾”。

实现上，我们用了 MCP 的一个 filesystem server 来读写这些 JSON，同时在 Agent 指令里强制要求：如果用户对输出表示修正，必须在最终回复前调用 memory update 工具，将修正意图写入 task memory。

### 第三层：Session Scratchpad（会话草稿）

这是短期的工作区，用于在一次长对话或多步推理中避免遗忘中间约束。比如你让 Agent 先收集资料，再写大纲，再逐段生成，每一步都可能产生新的约束（“第三段不要提某公司”），这些约束写入 Scratchpad（只是一个临时 JSON 或者甚至就是 Agent 内部变量），在后续步骤作为输入引用，任务结束后丢弃。

此层不需要持久化，但能显著减少“说后忘前”的问题。

## 操作步骤（以 OpenClaw 为例）

1. **建立 Profile 文件**  
   在 OpenClaw 后端能访问的目录放置 `profile.yaml`，确保有 MCP filesystem server 能读写。

2. **编写 MCP 记忆工具**  
   至少三个：
   - `read_profile`
   - `update_profile`（接受键路径和新值）
   - `update_task_memory`（接受 task_name、feedback 字符串）

   工具描述必须精确，让模型知道何时调用。例如 `update_task_memory` 的描述：
   > 当用户对任务输出给出纠正性反馈时，必须调用此工具记录反馈，用于未来相同任务的执行。

3. **改造系统指令**  
   去掉写死的风格要求，改为：
   > 任务开始时读取 user_profile。如是已知任务类型，读取对应 task_memory。生成输出前，应用所有偏好与历史修正。如用户给出纠正，必须调用 update_task_memory 后再回复。

4. **增量测试**  
   第一次运行某个任务，故意给出不符合偏好的输出，观察用户纠正后，下次是否改进。不能期待一次就完美；记忆系统是靠反复修正收敛的。

## 踩坑点

- **工具调用与回复的时序**：很多模型会先回复用户，再调用工具，导致修正已经发出去了才更新记忆。要在指令里明确“如果涉及更新记忆，先调用工具，再回复用户”。这需要对 Agent 循环有控制力，OpenClaw 里可以通过调整 `max_turns` 和指令强调来解决。
- **记忆膨胀**：feedbacks 无限堆叠会让 task memory 过大，后续上下文可能溢出。必须设置截断策略，比如只保留最近 5 条反馈并做摘要压缩，或让模型自己总结成一个新的 `preferred_rules`。
- **Profile 冲突**：通用 profile 与具体 task memory 冲突时（比如 profile 说 no emoji，但某个任务需要 emoji 分级标记），优先级要明确，通常 task memory 覆盖 profile。
- **多用户场景**：OpenClaw 个人使用一般单用户，如果有共享 Agent，必须按用户 ID 隔离 Profile。

## 可复用建议

1. **先把记忆建模为人可编辑的文件**：YAML/JSON，而非向量数据库。可读性允许你手动修正，也方便调试。
2. **记忆更新要走显式工具调用**，不要依赖模型“记住上次对话”，那是不稳定的。
3. **从 Task Memory 起步**，Profile 是更高层抽象，可以在有 3-5 个重复任务后再提炼。
4. **测试时用脚本自动重置记忆**，方便回归比较。
5. **与定时任务结合时，记得加载记忆的步骤**，很多人 cron job 里的 Agent 忘了读取记忆文件，结果每次都是默认风格。

## 总结

让 AI 助手记住你的偏好，不是靠更好的提示词，而是靠 **结构化记忆的工程流水线**：将偏好从自然语言中提取出来，写入可持久化的数据层，并在每次任务开始时可靠地加载并注入上下文。三层记忆模型（Profile、Task Memory、Session Scratchpad）是一个轻量、可落地、与 OpenClaw/MCP 生态契合的方案。经过几次修正迭代，Agent 的输出会逐渐收敛到你真正要的样子，而不是每次都重新谈判。

最终目标不是“像一个真朋友那样记住你”，而是 **在自动化和工具链里，消灭重复的沟通成本**。

---

