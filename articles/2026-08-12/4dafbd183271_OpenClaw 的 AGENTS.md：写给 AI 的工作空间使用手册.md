---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 32811
source: 综合讨论
publishedAt: 2026-08-12
---

## 为什么需要一个 AGENTS.md

OpenClaw 这类工作空间感知型 AI 代理，默认拥有读写文件、执行 Shell、调用 MCP 工具等能力。问题在于，它没有关于**你的项目应该如何工作**的上下文。缺少指导时，代理很容易用一个不被允许的全局安装修改系统环境，用不匹配的测试框架运行用例，或者在重构时意外删除业务逻辑。这些问题不是在模型能力层面发生的，而是在**工作空间约定缺失**这一层。

AGENTS.md 就是 OpenClaw 为此设计的固化提示文件：放在工作空间根目录下，由代理在会话初始化时自动解析并遵守。它相当于给一个经验丰富但完全不了解你代码库规则的新同事写一份“第一天入职须知”，只不过这个新同事会直接动手干活。

## AGENTS.md 的核心定位

与 `.cursorrules` 或 `.github/copilot-instructions.md` 不同，AGENTS.md 面向的是 **OpenClaw 代理的完整执行循环**。它不仅是补全时的风格提示，还会影响：

- 对 `tool-calls` 的筛选与参数生成
- 文件修改前的检查步骤（强制先读后写）
- 对危险的系统命令的拒绝边界
- 与 MCP 资源的交互方式（如数据库查询是否要走只读副本）

因此 AGENTS.md 需要写得比代码风格文件更具**动作性**和**安全约束性**。

## 书写步骤与关键内容

### 1. 创建文件并放置正确位置

在项目根目录创建 `AGENTS.md`。OpenClaw 会在会话启动时扫描该文件。不需要额外注册，也不需要在每次对话中单独引用。需要注意的是，如果项目使用了 monorepo 结构，推荐在顶层放置一个总纲式的 AGENTS.md，并在子包中放置更细粒度的 `AGENTS.md`（代理会优先读取操作目录最近的）。

### 2. 组织有效指令的结构

避免长篇自然语言散文，代理更容易遵循的是**结构化、边界清晰**的指令。推荐分节格式：

```markdown
# AGENTS.md

## Persona
You are a senior developer working on project X. Always prioritize safety and existing project conventions.

## Safety Rules
- NEVER run `rm -rf` or any destructive command without explicit user confirmation.
- Any command that modifies system-level config (e.g., /etc, global packages) must be rejected.
- Before executing a shell command with side effects, describe the impact and ask for confirmation.

## Workflow Rules
- Before editing any file, you MUST read it first.
- When adding new dependencies, check package.json for existing versions and use the same major version.
- Use `pnpm` for all package management; `npm` and `yarn` are not allowed.
- Tests must be run with `vitest`, not `jest`. The command is `pnpm test -- --run`.

## Coding Conventions
- Use TypeScript strict mode. Prefer type over interface for data models.
- Follow the existing folder-by-feature structure; do not create top-level /utils.
- API routes must be snake_case, frontend components PascalCase.

## MCP & Tool Usage
- When querying the database via MCP, use the read-replica connection string provided in `.env.mcp`.
- Logging tools should comply with the structured JSON format used in `src/logger.ts`.
```

### 3. 验证代理是否遵守

初始化一个新的 OpenClaw 会话，给出一个触及约束的低风险任务，观察代理的思维链（如果支持查看）与工具调用。例如，故意请它“帮我安装一个临时命令”，检查是否触发了全局安装拒绝规则。如果代理无视某条规则，原因通常是指令过于模糊，需要把行为直接映射到可判定的条件上，比如将“尽量使用 pnpm”改成“如果检测到 npm install，必须先报错并停止”。

## 踩坑记录

- **指令互相矛盾**：AGENTS.md 里说“不要主动创建新文件”，但任务又隐含需要新建，代理会进入反复确认的死循环。避免此类矛盾的唯一方法是**先写好 AGENTS.md，再在真实任务中跑一遍**，剪掉冲突段落。
- **忽略长指令**：当总字数超过 800 词后，代理可能只捕获前半部分。此时需要将指令拆分为 `AGENTS.md`（核心安全与工作流）和 `CONTRIBUTING.md`（详细规范），并在 AGENTS.md 中写明引用：“For detailed code style, see CONTRIBUTING.md”。
- **monorepo 覆盖**：如果子目录有自己的 AGENTS.md，父级规则会被部分忽略，且合并逻辑不是深层合并。需要显式在子文件的顶部声明继承：“Inherits from root AGENTS.md with the following overrides: …”。
- **重载不生效**：修改 AGENTS.md 后，当前会话不会自动刷新。必须新开会话，或明确说“请重新读取 AGENTS.md 以更新你的规则”。

## 可复用的工程化建议

1. **模板化**：为团队维护一个 AGENTS.md 模板仓库，覆盖安全规则、包管理器、测试框架等通用部分，新建项目时直接拉取。
2. **分层管理**：将 AGENTS.md 分为 `always`、`per-context` 两层。always 部分固定安全规则；per-context 部分根据任务类型（开发/部署/数据迁移）动态拼接，通过会话启动时的 prompt 注入。
3. **与 pre-commit 校验结合**：AGENTS.md 中写明“所有改动后必须运行 lint-staged”，并在项目配置里有真实的 lint-staged hook。这样代理即使遗漏，提交时也会被拦截，形成双保险。
4. **持续迭代**：每次代理做出违反预期的行为后，先不要只修正结果，应该问：“我能否在 AGENTS.md 里增加一条规则，让它永远不会再犯这个错误？”

## 总结

AGENTS.md 不是另一个规则文件，而是**将团队的隐性知识显性化为代理可以程序式遵守的边界**。它比代码风格工具更软，但对工程稳定性影响更大。花 30 分钟认真写一份贴合自己项目的 AGENTS.md，可能比花半天调试 AI 乱改的代码，性价比高得多。

当 OpenClaw 在工作空间里拥有了这样一份“使用手册”，它就不再是一个容易被高估的自动补全，而更像一个遵守纪律的协作成员。

---

