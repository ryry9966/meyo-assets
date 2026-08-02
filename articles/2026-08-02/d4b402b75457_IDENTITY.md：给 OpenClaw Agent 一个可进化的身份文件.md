---
title: IDENTITY.md：给 OpenClaw Agent 一个可进化的身份文件
feedId: 31304
source: 综合讨论
publishedAt: 2026-08-02
---

# IDENTITY.md：给 OpenClaw Agent 一个可进化的身份文件

## 背景：Agent 像一个永远失忆的新同事

用过 OpenClaw 或类似 Agent 框架做长周期自动化的人，大概都经历过这种场景：

昨天调好的 “日报助手”，你让它记住你喜欢表格大于段落、默认发送到某个群聊；今天重启会话后，它又给你生成整段文字发给了错误的人。不是因为模型能力退化，而是它根本不知道 “你是谁”、 “你们之间发生过什么”。

多数 Agent 实现靠 `system prompt` 注入角色设定，但 system prompt 是静态的：要么写死在一开始，要么靠模板变量动态拼。真正能跨会话记住偏好、积累项目经验的机制很少。一些方案引入了外部向量存储（RAG），但检索噪音大，且很难让 Agent 自主决定 “什么值得记住”。

OpenClaw 提供了一个更工程化的轻量解法：`IDENTITY.md`。一个站在 Agent 视角的、可被 Agent **自我维护** 的 Markdown 身份文件。

## 核心机制：不是 system prompt，是逐渐写满的笔记本

`IDENTITY.md` 不是一次性写好的角色说明书。它位于每个 OpenClaw workspace（或 agent 实例）根目录，工作流程大致是：

1. **启动加载**：OpenClaw 在每次会话初始化时，会把这个文件内容作为上下文注入（类似于项目里的 `CLAUDE.md` 或 `cursorrules`）。
2. **运行时参考**：Agent 在交互过程中能随时 “回想” 文件里已有的记录，比如用户偏好、项目结构、上次未完成的任务。
3. **会话结束反哺**：完成一轮工作或定时触发时，Agent 被要求 **总结本次交互中值得保留的信息**，并以合理格式追加／更新到 `IDENTITY.md`。

这样一来，Agent 的 “人格” 不再只依赖最初的 prompt，而是可以随着时间成长。本质上是一个 **自主维护的长期记忆文件**，不需要外部数据库，逻辑完全可控。

## 实操步骤（以 OpenClaw + MCP 工具链为例）

### 1. 启动脚手架
在一个干净的 OpenClaw 项目里创建初始 `IDENTITY.md`，写一个最小骨架：

```markdown
# Identity

## Who I am
I am an automation assistant for the data-eng team.

## What I know about my user
- (to be filled)

## Project context
- Project name: pipeline-monitor
- Stack: Python, ClickHouse, Docker

## Recurring preferences
- Communication: concise, bullet points, no emoji

## Log of important decisions
<!-- Agent will append here -->
```

关键是 **预先划分好区块**，避免 Agent 后续写入时破坏整体结构。

### 2. 配置 OpenClaw 加载策略
在 `claw.yaml` 或对应的 config 中指定加载逻辑：

```yaml
agent:
  identity:
    file: IDENTITY.md
    load_on_start: true
    update_after_session: true
    update_trigger: "every 10 turns"
```

`update_after_session` 和 `update_trigger` 控制了更新频率。建议初期使用 `every 10 turns` 触发，防止过多写入。

### 3. 给 Agent 写更新指令
在 system prompt 或工具描述中加入明确指引，例如：

> 每次触发自我更新时，先阅读当前 IDENTITY.md 的内容，然后在 Log of important decisions 下追加一段本次学到的东西（用户偏好、项目变更、执行过的命令结果摘要）。不要删除历史记录，除非明显过时。

### 4. 防膨胀与清理
设定文件大小上限（比如 800 行），超出时通知用户并要求人工归档。可以在 Agent 的 system prompt 里加入：

> 如果你发现 IDENTITY.md 超过 800 行，请暂停写入并提醒用户清理过时信息。

## 踩坑点

- **Agent 乱写格式**：最常出现。模型输出很容易破坏原有 Markdown 层级，尤其是列表和代码块。必须用少量示例（few-shot）喂给它更新格式，并且在更新逻辑中加入 `pre-commit` 检查（可以用一个简单的 lint 工具：判断 H2 标题数量是否减少）。
- **敏感信息泄露**：Agent 可能会把 API key、数据库密码写入 identity 日志。需要定期 grep 检查，或者在更新后由操作者做一次简单审计。
- **并发写入冲突**：多个会话同时触发更新会导致文件损坏。OpenClaw 默认加文件锁，但如果跑在分布式环境下，**尽量使用单一会话实例处理 identity 更新**，其他会话设为只读。
- **身份塌缩（Identity collapse）**：不加节制地追加会让文件变成冗长的流水账，反而降低推理质量。解决方法是要求 Agent 在更新时做 **合并与过期判断**，例如 “用户最近三次交互都要求中文回复，之前英文偏好可以标注为 outdated”。

## 可复用建议

1. **模板化区块**：保持 `Who I am`、`Project context`、`Preferences`、`Decisions log` 四大模块，Agent 只能在 `Decisions log` 自由追加，其余部分人工修改。
2. **版本控制**：把 `IDENTITY.md` 纳入 git 管理，每次更新后自动 commit，方便回滚。
3. **与 MCP 工具联动**：比如让 Agent 在发现某条信息特别重要时，写到 identity 的同时，通过 MCP 的 `obsidian` 或 `notion` tool 做一份备份。
4. **人工定期校准**：每周花 5 分钟读一遍 identity，把 Agent 自己学到的错误信息删掉，这比调 prompt 更实在。

## 总结

`IDENTITY.md` 给 Agent 工程增加了一层 “记忆体”。它不解决所有问题，但对于那些需要 Agent 在不同时间段稳定为你工作的场景（日报、监控、定期报告），价值很大。

把 Agent 当成一个可以写工作日志的同事，而不是每次都从空白开始的对话窗口，思路会清晰很多。

如果你还在用纯 stateless prompt 拼装身份，不妨从这个单文件开始。

---

