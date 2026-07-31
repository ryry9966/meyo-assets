---
title: 用 IDENTITY.md 实现可进化 Agent 身份：OpenClaw 的持久化自我修正方案
feedId: 31103
source: 综合讨论
publishedAt: 2026-07-31
---

# 用 IDENTITY.md 实现可进化 Agent 身份：OpenClaw 的持久化自我修正方案

## 背景

在基于 OpenClaw 构建的智能体（Agent）实践中，一个高频痛点很快浮出水面：**系统提示词（system prompt）是死的，但任务场景是活的。**  
无论是通过 MCP 工具链扩展能力，还是依赖插件自动化执行流程，Agent 的行为模式通常被硬编码在配置中。一旦部署，修正它的“认知”只能靠开发者手动改提示词，导致：

- 用户反馈无法沉淀为 Agent 的长期记忆；
- 经过多轮调试得出的有效思维框架，无法被 Agent 自己记住并复用；
- Agent 在面对连续性任务时，缺乏从过往经验中自我调整的能力。

OpenClaw 社区中逐渐出现一种轻量级的解决方案——**IDENTITY.md**。  
它把 Agent 的身份定义、核心目标、经验法则、个性化偏好统统放进一个 Markdown 文件，并允许 AI 在特定条件下**主动修改该文件**，从而实现身份的“进化”。

## 问题拆解

本质上，我们需要的是一个**可读写的外部记忆体**，而非一次性注入的静态 prompt。  
这个记忆体需要满足几个工程约束：

1. **结构化可解析**：AI 能可靠地读取、定位并更新特定段落；
2. **可审计与回滚**：修改历史可追溯，防止身份被污染后无法复原；
3. **权限可控**：避免 Agent 随意篡改造成不可预期的行为漂移；
4. **跨会话持久**：重启或新对话后，身份状态不丢失。

OpenClaw 自身提供了**文件系统 MCP 工具**与**工作区隔离**，结合 Git 版本控制，使得 IDENTITY.md 方案天然成立。

## 做法与步骤

### 1. 创建初始 IDENTITY.md

在 OpenClaw 工作区根目录新建 `IDENTITY.md`，按如下最小模板填写：

```markdown
# Identity

- **Name**: OpenClaw Assistant
- **Core Goal**: Help the user automate development workflows
- **Principles**:
  - Prefer explicit instructions over guessing
  - When uncertain, ask before acting

# Experience Log

<!-- Last updated: 2025-03-01 -->
- [2025-02-28] User prefers Jest over Mocha for testing.
- [2025-03-01] Learned: when deploying, always check .env.example changes first.

# Evolving Rules

- If a task fails 3 times, escalate to manual mode.
- Never modify config files without dry-run output.
```

关键点：**将可变部分（Experience Log、Evolving Rules）与稳定部分（Core Goal/Principles）用不同章节显式分开**，方便 AI 精准更新。

### 2. 暴露文件读写能力

在 OpenClaw 的 MCP 配置中，确保 Agent 有权限读取并写入工作区内的 `IDENTITY.md`。  
典型配置段（示意）：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/workspace"],
      "allowedOperations": ["read", "write"]
    }
  }
}
```

**安全提醒**：将 allowed paths 限制在工作区目录，防止 Agent 误触系统文件。

### 3. 设计更新触发机制

不要每轮对话都允许修改身份文件，否则很快被噪声淹没。建议采用以下触发策略之一：

- **显式命令**：用户输入 `/learn` 或 `/remember` 时，Agent 追加经验条目；
- **任务后总结**：在插件执行完成后，由 Agent 自动判断“是否有值得记录的教训”，再调用文件写入工具追加一行；
- **阈值触发**：例如同类错误出现 N 次，Agent 主动在 Evolving Rules 中添加防范规则。

在系统提示词中加入指令：

> When you finish a task, check if any new insight should be recorded in `IDENTITY.md` under `Experience Log`. Update it using the file tool, keep entries concise, and never remove existing user preferences.

### 4. 增量更新与结构化写入

直接用文件覆盖的方式风险极高。推荐做法：

- 先用 `read` 工具读取整个文件；
- 在文本中定位到对应章节（以 `## Experience Log` 为锚点）；
- 构造新的条目，用正则或字符串替换插入；
- 调用 `write` 工具写回。

为保证解析鲁棒性，可以在文件中加入特殊注释作为锚点标记（如 `<!-- EXPERIENCE_LOG_START -->`），避免章节标题变更导致插入错位。

## 踩坑点

1. **全量覆盖风险**：  
   如果不限制 AI 的写能力，它可能用不完整的片段覆盖整个文件，导致身份信息被截断。必须通过提示词严格约束“只追加，不删除现有用户偏好”。

2. **多会话竞态**：  
   如果同一个 Agent 被多个会话并发读写同一文件，可能产生冲突。建议在文件开头设置 `session_id` 或时间戳锁，或限制一次只允许一个活跃会话更新。简单做法：依赖文件系统锁，或使用 Git 检测冲突。

3. **自我强化循环**：  
   Agent 可能将错误经验写入规则，而下次又遵循该错误规则，导致行为劣化。需要在 Evolving Rules 中加入“可验证”的描述，并要求 Agent 在添加规则时标注出处和验证条件。定期人工审查 `git diff`。

4. **权限泄露**：  
   如果 MCP 文件系统权限未严格限定，Agent 可能读取或篡改其他文件。始终将 allowed paths 限制在独立工作区，并禁止访问系统路径。

## 可复用建议

- **纳入版本控制**：  
  在 OpenClaw 工作区初始化 Git 仓库，`IDENTITY.md` 每次修改后自动提交（可借助插件），这样任何身份漂移都可通过 `git log -p -- IDENTITY.md` 回溯。

- **模板化分段**：  
  除了固定章节，可增加 `# Preferences` 记录用户工具链偏好（如包管理器、测试框架），`# Anti-Patterns` 记录禁止的行为，形成可复用的身份模板。

- **测试身份复原**：  
  编写一个小型集成测试：给定一组任务，检查 Agent 是否能根据 IDENTITY.md 中的经验调整行为。适用于 CI 流程，确保身份演进不破坏核心功能。

- **人工巡检节奏**：  
  即使自动化，也建议每周抽检 `IDENTITY.md` 的 diff，清理无效条目，合并冗余规则。可设置定时任务将变更推送到私有的 review 分支。

## 总结

IDENTITY.md 本质上是一种 **Agent-native 的元记忆基座**——它用最朴素、可读、可审计的 Markdown 文本，解决了智能体“学到的东西去哪儿了”的问题。  
它不依赖外部向量数据库，不需要复杂的记忆管理框架，完全基于文件系统与 MCP 工具，与 OpenClaw 的工作流机制天然契合。

在实际工程中，将其视为一个**自我修正的配置层**，而非单纯的 prompt 文件。当你开始把 Agent 的思维框架、经验教训、用户偏好都持续写入这一个文件中时，你得到的就不再是一个静态的工具，而是一个**随任务成长的工作伙伴**。

---

