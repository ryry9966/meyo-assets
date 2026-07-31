---
title: 让 Agent 主动做事：用 MCP 搭一个 GitHub 讨论自动巡检助手
feedId: 31126
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：为什么需要 proactive 能力

大多数 AI 助手都是被动式的：你发一条消息，它回一条结果。当你需要它规律性地检查外部状态、或是在某个条件满足时自动介入，传统的“一问一答”模型就会显得很吃力。

在工程团队里，有一类典型场景：开源维护者每天要扫一遍 GitHub Discussions / Issues，判断哪些是新问题、哪些需要跟进。这件事本身不复杂，但重复且消耗注意力。如果能有一个 Agent 每天定时拉取未回复的讨论，生成一份草稿回复或摘要，开发者只需要在上班前快速扫一眼、确认发送，效率会高很多。

这就是“proactive”能力：不等你开口，它已经把活儿干完了，等你验收。

## 问题拆解

我们要实现这样一个 agent：

1. **定时触发** — 每天早上 8 点自动执行一次；
2. **获取数据** — 从 GitHub 拉取某个 repo 的 discussions，过滤出没有维护者回复的；
3. **分析内容** — 理解 discussion 的意图（bug 报告？功能请求？使用疑问？）；
4. **生成草稿** — 给出一个符合项目语气的模板回复，附上可能的参考链接；
5. **输出结果** — 将草稿保存为本地 markdown 文件或发送到飞书/Slack 通知，方便人工确认。

整个链路中，“主动”体现在步骤 1 的定时触发，其余部分都可以用已有的 MCP 工具和 LLM 组装。

## 工具选型与环境

- **Agent 框架**：使用 OpenClaw 作为调度核心（支持 cron 触发器和插件系统），也可以换成任何支持定时任务 + function calling 的 agent 运行时。
- **数据源**：GitHub GraphQL API，通过 `@anthropic/mcp-server-github` 这个 MCP 服务器接入，避免重复造 token 管理和接口拼装的轮子。
- **LLM**：Claude 3.5 Sonnet，负责讨论分类与回复生成。
- **通知**：可选的 Slack Webhook / 本地 markdown 写入。

## 实现步骤

### 1. 配置 GitHub MCP 服务器

先在本地或容器里启动 `mcp-server-github`，需要一个具有 `read:discussions` 权限的 personal access token。MCP 配置示例（简略）：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your_token>"
      }
    }
  }
}
```

启动后 agent 就能通过 MCP 协议调用 `search_repositories`、`list_discussions`、`get_discussion` 等工具。

### 2. 编写定时巡检任务

在 OpenClaw 中创建一个 cron 技能，每天早上 8 点触发。关键步骤是调用 MCP 工具分页获取指定 repo 的 discussions，并筛选出 `answerChosen: false` 且 `comments` 中没有作者以外用户回复的条目。

因为 OpenClaw 支持将工具调用直接组合为任务，我们只需要用自然语言描述流程，由 agent 自己调用 MCP：

> “每天早上 8 点，检查 openclaw/community 仓库的 discussions。列出所有没有 project member 回复的讨论，提取标题、作者、正文前 500 字，保存为 /reports/discussions_$(date).md。”

但你也可以显式写成一个 function chain 以保证稳定性：先 `list_discussions`，拿到 ID 列表，再用 `get_discussion` 逐个获取详情，写过滤逻辑。最后把筛选结果传给 LLM 做分析和生成。

### 3. 分析讨论意图并生成草稿回复

将筛选出的每条讨论 body 传给 LLM，要求输出结构化 JSON：

```json
{
  "category": "bug|feature_request|question|other",
  "summary": "一句话总结",
  "draft_reply": "基于项目语气的草稿回复，不超过 3 句话",
  "suggested_labels": ["bug", "needs-triage"]
}
```

注意 prompt 中要注入项目风格手册（CONTRIBUTING.md 摘要）和已有回复的示例，避免生成过于热情或过于冷漠的机器人口吻。

### 4. 输出与确认环节

生成结果后不直接发到 GitHub（危险操作），而是写入本地 reports 目录或通过 Webhook 发送到团队群。每条草稿前标明编号，方便团队成员在早会上快速决策：“这条我认领，直接发送；那条需要先澄清。”

## 踩坑点

**a) API 限流与分页陷阱**
GitHub GraphQL API 有严格的 rate limit。MCP 服务器默认不会自动处理复杂的游标分页，如果是大型项目，一次性拉取全部 discussions 容易触发限流。需要加缓存：每天只请求上次检查点之后更新的讨论，用 `updatedAfter` 参数做增量拉取。

**b) 讨论 vs Issue 的语义混淆**
Discussions API 返回的 `category` 是项目自定义的，不一定和 Issue 标签对应。直接套用 Issues 的分类逻辑会导致大量 “other” 分类。需要先跑一遍现有 discussions，总结出实际出现的 category 名称集合，然后写进 prompt 映射表。

**c) LLM 幻觉与“过度 proactive”**
让 LLM 直接生成可发送的回复，容易产生虚构的解决方案链接、承诺未实现的功能。安全做法是：强迫它在草稿中引用的任何链接都必须来自讨论内已出现的引用或 repo wiki/文档，禁止自行编造。可以通过输出校验（正则匹配链接域名）做一层防御。

**d) 私密讨论的权限问题**
如果 repo 包含内部团队讨论（非公开），MCP token 的权限必须严格限定为只读，且草稿存储位置要避开公开网盘。主动系统在数据安全上的攻击面比被动系统大得多——每多一条自动拉取数据，就多一个泄露点。

## 可复用建议

1. **把 proactive 任务拆成“感知-决策-执行”三层**  
   感知层只负责稳定数据接入（MCP 工具），决策层做 LLM 分析并输出结构化结果，执行层保留人工确认或简单自动化（归档、打标签）。三层解耦后，复用和调试代价都低很多。

2. **先跑“干跑模式”一周**  
   让 agent 每天拉数据、生成草稿，但全部存在本地且不发送任何通知。一周后人工复查准确率，调整 prompt 和过滤条件，再接入通知渠道。

3. **用结构化输出兜底**  
   让 LLM 返回 JSON 而非 Markdown 自由文本，可以减少解析错误。同时能给每个字段加上校验规则（比如 `draft_reply` 长度限制、必含引用来源），提升可靠性。

4. **记录每次运行的成本与耗时**  
   Proactive 系统是“持续运转”的，token 消耗容易被忽视。最好在日志里记录每次执行的 prompt token 数，过几周就能看出是否需要做增量同步或缓存 summary。

5. **加入“主动暂停”机制**  
   如果你的巡检 agent 突然发现一晚上多出 50 条未回复讨论（很可能是 spam 或事件爆发），应触发保护逻辑：暂停自动生成，只通知原始列表，等待人工介入。否则容易生成大量无意义草稿，淹没真正重要的信息。

## 总结

Proactive 不等于“替你决策”。工程上最可靠的 proactive AI，是那种把脏活累活提前干完、把“要不要发”这个按钮留给你的人机协作模式。

本文给出的 GitHub 讨论巡检方案，本质是一个“定时感知 + LLM 分析 + 人工审批”的轻量工作流。通过 MCP 接入外部数据源，加上 Agent 的 cron 能力，一天就能搭出来。实际跑起来后，你会发现自己每天早上面临的不是一堆待读通知，而是一份整理整齐的草稿单——这时候 proactive 的价值才真正体现。

---

