---
title: 给 OpenClaw 写一份靠谱的 AGENTS.md — 让 Agent 按你的规矩来
feedId: 32545
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：Agent 没你想的那么自觉

在 OpenClaw 的多 Agent 协作场景中，最常见的翻车现场不是模型不够聪明，而是模型太“自由”。每次对话你都要重复一遍：

- 当前工作目录是哪
- 可用的 MCP 服务器有哪些
- 哪些插件该用、哪些不该碰
- 输出格式是 JSON 还是 Markdown

一旦上下文窗口滚动，这些约束就逐渐被遗忘。更麻烦的是，当你有一个固定项目结构时，Agent 常会“发明”不存在的路径，或者调用根本没配置的工具。

OpenClaw 给出的解决方案很直接：**AGENTS.md** — 一份放在项目根目录（或通过配置指定）的 Markdown 文件，作为 Agent 的工作空间使用手册。它会被注入到系统提示中，持久生效，任何人切换到该工作空间都能获得相同的 Agent 行为基线。

## 问题：没有规范手册的后果

我们把常见痛点收敛成三类：

1. **上下文漂移**  
   长对话中 Agent 忘记最初设定的工具白名单，开始调用错误的 MCP 服务器，甚至用“幻想”结果替代真实返回。

2. **指令碎片化**  
   规则散布在 `openclaw.yaml` 的 prompt 段、代码注释、对话开头，维护困难，交接给其他人时完全不可复现。

3. **调试成本高**  
   每次排查行为偏差，都需要扒开长长的系统提示，猜测到底哪条规则没生效或相互冲突。

本质上，我们缺的不是 prompt，而是一个结构化的、可版本控制的“工作空间规格书”。

## 做法：从零写一份 AGENTS.md

### 1. 基础骨架：三个必须回答的问题

OpenClaw 在加载 Agent 工作空间时，会把 AGENTS.md 的全文拼入系统指令，因此文件本身需要自包含。我实践的骨架长这样：

```markdown
# Workspace: your-project-name

## 1. What is this workspace for?
用 2-3 句话描述当前工作空间的职责边界。
例如：“这是一个 Python 后端项目的工作空间，负责 API 开发、数据库迁移和本地测试。”

## 2. Where is everything?
- Project root: /abs/path/to/project (Agent 可读取此环境变量)
- Source code: src/
- Config: config/
- Docs: docs/
- Tests: tests/
```

### 2. 工具与插件声明

告诉 Agent 工具箱里有什么，以及何时使用：

```markdown
## 3. Available tools & when to use them
### MCP Servers
- **filesystem** — 所有文件读写必须通过此服务器，不允许在对话中直接虚构文件内容。
- **playwright** — 仅用于 e2e 测试场景，禁止在普通 API 调用中使用。
- **database** — 连接本地测试库，只读操作必须走此服务器。

### Plugins
- **shell-command**: 允许执行 `pytest`, `ruff`, `mypy` 等检查命令。
- **code-review**: 对 PR 进行自动化 review 时使用，勿用于一般聊天。
```

关键点：**每个工具附上使用时机（when）**，而不是只列出名字。这能解决大量误调用。

### 3. 行为约束与输出规范

Agent 不仅要“会”做事，还要“像团队的一员”。这部分写到操作粒度：

```markdown
## 4. Rules of engagement
- 修改文件前，先输出修改计划并获得确认。
- 测试失败时，先分析日志，不要直接重写测试。
- 禁止使用 `echo` 或 `print` 调试，必须使用项目配置的 logging。
- 输出代码时默认使用 Markdown code block 并带语言标识。
- 涉及数据库操作，必须在回复末尾附上 migration 编号。

## 5. Response format
- 任务完成：`[DONE] 总结结果`
- 需要确认：`[CONFIRM] 具体待确认项`
```

这些规则被评价为“啰嗦”，但在多人多 Agent 环境中，这是唯一靠谱的胶水层。

### 4. 与 openclaw.yaml 的协同

AGENTS.md 不是孤立文件。在 `openclaw.yaml` 中这样引用：

```yaml
workspace:
  name: my-project
  agents_md_path: ./AGENTS.md
  # 亦可直接写为默认的 AGENTS.md，OpenClaw 会自动发现
```

我会额外在 `openclaw.yaml` 的 `agent.custom_prompt` 中加一条简短的引用指示：

> “Your workspace manual is provided in AGENTS.md. Always follow the rules there unless explicitly overridden by the user in this conversation.”

这样在紧急覆盖场景下，用户指令优先，但默认基线仍是 AGENTS.md。

## 踩坑点

1. **路径硬编码**  
   AGENTS.md 里写死绝对路径 `/home/alice/projects/foo` 是常见错误。别人的机器或 CI 环境路径不同，导致 Agent 寻址失败。解决方案：统一使用环境变量 `$PROJECT_ROOT` 或相对路径，并在 AGENTS.md 中声明“所有路径以 project root 为基准”。

2. **过度约束导致 Agent 死板**  
   把 AGENTS.md 写成操作手册的逐行步骤，Agent 会严格执行，一旦出现预期外的分支就卡住。应保留“常识性”空间，例如写“优先使用 MCP filesystem 读写文件，特殊情况可向用户确认”，而不是“永远禁止使用内置文件工具”。

3. **更新滞后**  
   项目引入了新的 MCP 服务器、插件，但没更新 AGENTS.md，导致 Agent 使用过时的工具列表。建议在 CI 中加入一个简单的 lint 检测：对比 `openclaw.yaml` 中声明与 AGENTS.md 中的工具/插件一致性。

4. **编码与换行符**  
   曾经踩过一个坑：Windows 开发环境生成的 AGENTS.md 含有 CRLF，在 WSL 中被 OpenClaw 加载时，换行被解析为多余的空行，影响提示注入。统一要求 UTF-8、LF。

## 可复用建议

- **分层结构**  
  公共团队 AGENTS.md（团队通用规则） + 个人覆盖文件。OpenClaw 支持通过 `include` 语法合并多个提示文件，可把通用部分抽离成 `AGENTS.base.md`，再按需拼接。

- **模板化**  
  把项目特定的路径、版本号用占位符 `{{PYTHON_VERSION}}` 标记，通过 CI (如 `envsubst`) 生成最终版 AGENTS.md，避免人工修改。

- **在 README 中关联**  
  对于新加入的开发者，在项目 README 中明确：“本项目的 AI Agent 使用 AGENTS.md 定义工作空间行为，修改规则请提交 PR 并 @你的 AI 同事”。

- **定期审查**  
  每两周在团队例行巡检中检查一次 AGENTS.md，看是否有过时规则、新增工具未注册，确保与真实工作流一致。

## 总结

AGENTS.md 不是一段花哨的 prompt 炫技，它更像是给 AI Agent 配发的“工作证”加“巡检单”。它让 Agent 从“一次性临时工”变成“可复现的协作单元”。在 OpenClaw 社区里，凡是多 Agent 稳定运行的项目，几乎都有一份维护良好的 AGENTS.md。

在自动化实践中，最值得投入的时间不是调参，而是写清楚规则 — 因为 **规则是唯一不需要算力就能提升 Agent 产出的东西**。

---

