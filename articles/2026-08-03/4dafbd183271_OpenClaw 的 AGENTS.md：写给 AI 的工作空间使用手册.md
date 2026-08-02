---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 31410
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：当 AI 代理走进你的项目

过去一年，AI 编码代理（如 OpenClaw 内置的 Agent 运行时）从“能跑就行”走向了生产环境的日常协作。它们可以直接调用 MCP 工具、修改文件、执行命令，甚至提交代码。但一个尖锐的问题也随之暴露：**代理对项目的隐性约定一无所知**。

人类开发者有 onboarding 文档、团队规范、口头传承，而代理只有一个孤立的文件树和工具列表。于是你会看到：它用错了包管理器、忽略了测试框架的特殊参数、在禁止直接提交的分支上执行了 `git push`，或者反复调用某个 MCP 服务器却始终拿不到有效结果——因为它根本不知道该用哪个工具。

OpenClaw 的回应是 **AGENTS.md**——一个放在工作空间根目录、专为 AI 代理编写的“使用手册”。它不是 README，不是 CONTRIBUTING，而是**结构化的指令集**，直接注入代理的上下文，约束和引导其行为。

## 核心问题：隐性知识如何显式化

没有 AGENTS.md 时，代理的决策完全依赖基座模型的通用推理和非常有限的用户提示。这会引发三类典型故障：

1. **环境误配**：代理默认使用 `npm`，而项目强制用 `pnpm`；或在未激活 venv 的情况下执行 Python 脚本。
2. **工具滥用**：MCP 挂载了五个服务器（Jira、Confluence、内部 API、Slack、数据库），但代理遇到搜索文档需求时，可能随机选择一个，或者全部遍历一遍，浪费 token 和时间。
3. **流程破坏**：代理为修复一个 lint 错误直接 push 到 main，而没有走 feature branch + PR 的约定。

这些问题无法通过“更好的 prompt”彻底解决，因为每次对话的 prompt 往往只有任务描述，缺少对工作空间运行时的持续记忆。AGENTS.md 扮演的就是这个**持久化、可版本化的系统指令**。

## 做法：编写有效的 AGENTS.md

### 1. 文件位置与加载机制
OpenClaw 启动 Agent 对话时，会自动扫描工作空间根目录的 `AGENTS.md`（可在 `.occonfig` 中通过 `instructionsFile` 重写路径）。加载后，内容被注入到系统提示的顶部区域，权重高于内置身份声明。**注意：目前不会自动重载，如果中途修改了文件，需要显式使用 `/reload` 或开启新会话**。

### 2. 推荐内容结构
一个生产可用的 AGENTS.md 应包含以下区块，按优先级排列：

```markdown
# Project Context
- This is a Python monorepo using `uv` for dependency management.
- All services are in `services/`, shared libs in `libs/`.
- Python version: 3.11; line length: 100 (black/ruff).

# Environment & Commands
- Activate environment: `source .venv/bin/activate` (Linux/macOS) or `.venv\Scripts\activate` (Windows)
- Install dev deps: `uv pip install -r requirements-dev.txt`
- Run tests: `pytest -x --cov=src --cov-report=term-missing`
- Lint: `ruff check . && black --check .`

# MCP Tool Usage
- Use `mcp__confluence_search` **only** when asked to find internal documents. Prefer `space="ENG"` and filter by `updated > 2024-01-01`.
- Never call `mcp__slack_post` without explicit user confirmation.
- For database queries, use `mcp__readonly_db`—the write-enabled MCP is OFF BY DEFAULT.

# Coding Standards
- Prefer `dataclasses` over plain dicts for internal DTOs.
- Any new endpoint must have an OpenAPI decorator and a dedent docstring.
- Do not add type: ignore unless reviewed.

# Workflows
- Before committing: run `pre-commit run --all-files`; only commit if it passes.
- Branch naming: `<type>/<ticket-id>-short-description`, e.g., `feat/ABC-123-add-cache`.
- Push to origin only after creating a draft PR branch.
```

### 3. 与 MCP 工具解耦的技巧
不要在 AGENTS.md 里硬编码凭证或端点，这会把安全信息泄露给模型（且版本控制下很危险）。正确做法是：描述**何时、用什么条件选择工具**，具体连接参数留在 MCP 配置的 `server.json` 中。例如：

```markdown
# Good
When user asks about incident history, search `mcp__pagerduty_incidents` with status=resolved, limit=5.
# Bad
Use API key sk-xxx and endpoint https://...
```

### 4. 动态指令与用户意愿的共存
有时用户会在对话里临时要求“忽略测试直接提交”。此时代理需要服从最新指令，但 AGENTS.md 的约束会形成阻力。为了解决冲突，可以在文件末尾添加一条优先级规则：

```markdown
# Precedence
User instructions in the current conversation always override the rules above, except when explicitly prohibited by a higher-level policy (e.g., never delete production data).
```

这给代理一个明确的冲突解决策略，减少反复确认。

## 踩坑实录

- **AGENTS.md 过长**：早期团队把整个编码规范（上千行）塞进去，导致系统提示 token 占用甚至超过了上下文窗口的 40%。代理开始“遗忘”对话中后段的任务。**解法**：保留核心决策规则，细节通过 `@include` 拆分到 `docs/agents/` 下，并用工具检索（例如当代理需要具体规则时，主动检索 docs）。
- **路径硬编码**：写了 `cd /home/bob/project`，但 CI 环境路径不同，代理执行失败。**必须使用相对路径或环境变量**。
- **禁忌词导致死循环**：要求“绝对不要运行 `git push --force`”，代理谨慎地反复自问“这是否属于 `git push --force`？”消耗大量 token。改为正面指令：“请始终使用 `git push` 默认选项，除非用户明确要求 force push 并给出理由。”
- **未同步更新**：项目迁移到 `rye` 管理后，AGENTS.md 仍指示用 `uv`，代理在两周内不断报错。**将 AGENTS.md 的更新纳入 Definition of Done**，并在 CI 中添加检查（如比对 husky/pre-commit 的锁定文件）。

## 可复用建议

1. **模板化起步**：新建项目时从 OpenClaw 社区模板库复制一份 `AGENTS.md`，包含常见命令、MCP 使用注解、安全边界。持续根据团队 feedback 裁剪。
2. **拆分大文件**：如果指令超过 300 行，拆成 `AGENTS.md` (核心规则) + `docs/agents/` 下分类文件，然后在核心文件里写：“For detailed testing guidelines, refer to `docs/agents/testing.md`； when needed, search that file.”
3. **与 MCP 工具清单联动**：写一个脚本，读取当前 `.occonfig` 中启用的 MCP 服务器列表，生成工具概览块，确保 AGENTS.md 里的工具名与实际注册名一致。
4. **版本控制与审查**：AGENTS.md 的变更应该走 PR 审查，因为它直接影响所有使用该仓库的代理实例。审查要点：是否有新增的破坏性命令、是否泄露密钥、指令是否模糊。
5. **定期演练**：每月用一个新的 OpenClaw 会话，只给一个模糊任务（如“加一个缓存层”），观察代理是否遵从 AGENTS.md。根据偏差迭代。

## 总结

AGENTS.md 不是“给 AI 写 README”，而是**把团队约定翻译成代理可执行的约束**。它解决了工作空间隐性知识丢失的问题，让 OpenClaw 代理从“随机应变的陌生人”变成“了解家规的协作者”。工程化的关键在于：保持短小、精炼、可验证，并将维护工作融入日常流程。

下次当你不小心在对话里反复纠正代理的行为时，不妨停下来想一想：这条规则，是不是应该写进 AGENTS.md？

---

