---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35509
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景：为什么需要一个给 AI 看的“使用手册”

OpenClaw 作为工作空间智能体，默认情况下每次新会话都像一个刚入职的临时工：它有能力，但不了解你的项目结构、命令习惯、环境边界。你可能遇到过这些情况：

- 在 monorepo 里直接跑根目录 `install`，而不是进入子包；
- 分不清 dev / staging / prod 环境变量，差点把生产配置改掉；
- 明明接好了 MCP 和插件，却乱调用工具，或者完全不敢调用；
- 每次开新会话都要重复解释“这个目录不能动”“测试命令是 xxx”。

这些问题的根源不是模型能力不足，而是缺少一个稳定、机器可读的工作空间约束文件。AGENTS.md 就是 OpenClaw 生态里用来干这件事的：它是一份写给 AI 的操作手册，而不是写给人类看的 README。

## 问题：无约束的 Agent 会带来什么

OpenClaw 支持 MCP、插件和自动化任务，能力越强，越需要边界。一个没有被约束的 Agent 通常会：

- 根据训练数据里的“通用项目习惯”猜测你的工作流，忽略你的特殊约定；
- 在错误的工作目录执行命令，生成路径错乱的产物；
- 把自动化任务当作“尽力而为”，不知道该在哪些步骤停下来等人工确认；
- 面对多个 MCP server 时无从下手，要么全部不调用，要么乱调一通。

这些行为会直接拖慢开发节奏，甚至造成文件覆盖、错误提交、生产环境误操作。与其每次口述规则，不如把规则写进一个文件，让 OpenClaw 每次启动时自动加载。

## 做法：从零开始维护 AGENTS.md

在仓库根目录创建 `AGENTS.md`，并纳入版本控制。OpenClaw 会在启动工作空间时自动读取该文件，并将其作为系统提示的一部分注入。文件内容建议按以下结构组织：

### 1. 项目概览
用 2-3 句话说明这个仓库是什么、主要语言、单体还是 monorepo、部署形态。不要写成长篇大论。

### 2. 目录约定
明确哪些目录是源码、哪些是生成物、哪些目录禁止修改。例如：

- `src/` 为手写源码；
- `dist/`、`build/` 为构建产物，不要编辑；
- `.github/workflows/` 为 CI 配置，禁止改动；
- `packages/shared/types/` 是公共类型，生成代码前先查看。

### 3. 常用命令
按场景列出命令，并标注允许自动执行还是需要确认：

- 开发启动：`pnpm --filter @app/web dev`
- 单元测试：`pnpm -r test`
- 构建检查：`pnpm -r build`
- 部署相关：仅允许 `deploy --dry-run`，真实部署必须人工确认

### 4. 环境变量与密钥
告诉 Agent 去哪里找环境变量示例文件（例如 `.env.example`），并明确：**不要把真实密钥写入 AGENTS.md 或代码中**。如果需要传密钥，通过工作空间的 secrets 管理或环境变量注入。

### 5. MCP / 插件使用说明
列出当前可用的 MCP server 和插件，并说明各自用途：

- `filesystem` MCP：可读写 `./workspace`、`./output`，其他目录只读；
- `playwright` MCP：仅允许访问测试环境域名，禁止访问生产；
- `github` MCP：只读仓库，禁止创建 PR 或修改 issue。

### 6. 自动化边界
用“允许 / 禁止 / 需确认”三段式写清：

- 允许：运行测试、格式化代码、创建新文件、读取日志；
- 禁止：修改 CI 配置、删除文件、 force push、修改依赖锁文件；
- 需确认：执行数据库迁移、发布构建、合并分支。

### 示例最小模板

```markdown
# AGENTS.md

## Workspace
- 本仓库为 pnpm monorepo，禁止使用 npm 或 yarn。
- 源码位于 packages/*/src，公共类型在 packages/shared/types。

## Commands
- dev: pnpm --filter @app/web dev
- test: pnpm -r test
- build: pnpm -r build
- deploy: 仅允许 --dry-run

## MCP / Plugins
- filesystem: 可写 ./workspace 与 ./output
- playwright: 仅测试环境域名
- github: 只读

## Guardrails
- 不要修改 .github/workflows 下任何文件。
- 生成代码前先检查 packages/shared/types 是否已有类型定义。
- 遇到需要真实部署的操作，必须停下来询问。
```

## 踩坑点：这些细节最容易翻车

1. **文件过长导致截断或稀释注意力**  
   OpenClaw 读取 AGENTS.md 有 token 成本。文件太长会被截断，或者重要规则被淹没。建议控制在 300-500 行以内，复杂子项目使用独立的 AGENTS.md 分层放置。

2. **路径写死、反斜杠混用**  
   Windows 环境下反斜杠容易引起解析问题。建议统一使用正斜杠或相对路径，避免绝对路径写死在文档里。

3. **把真实密钥写进 AGENTS.md**  
   AGENTS.md 会被 Agent 读取，也可能被日志记录或分享。**任何情况下都不要写入真实密钥、token、数据库密码**。

4. **与 README 混淆**  
   README 是给人类看的，可以写背景故事、徽章、截图；AGENTS.md 是给 AI 看的，要求短句、命令式、可执行。不要把大段散文复制进来。

5. **约束过松或过严**  
   禁止太多，Agent 会频繁停下来提问；禁止太少，Agent 会乱来。用“允许 / 禁止 / 需确认”分区，比单纯堆规则更有效。

6. **更新滞后**  
   过期的 AGENTS.md 比没有更危险，因为 Agent 会信任它。每次目录结构、命令、环境约定变化时，必须同步更新，最好在 PR checklist 里加一项。

## 可复用建议

- **将 AGENTS.md 纳入项目脚手架**：新项目初始化时自动生成最小版本，避免从零开始。
- **分层放置**：根目录放全局规则，子目录放局部规则，例如 `frontend/AGENTS.md`，适合大型 monorepo。
- **用“触发条件”代替描述性规则**：例如“当检测到 Python 文件时，先运行 ruff check”，比“请保持代码风格一致”更可执行。
- **与 MCP 工具联动**：在 AGENTS.md 中列出每个 MCP server 的预期输入输出，Agent 会更愿意调用合适的工具。
- **定期让 Agent 自检**：可以写一个一次性 prompt，让 OpenClaw 对比 AGENTS.md 与实际目录结构，输出不一致的 diff 建议，帮助维护。

## 总结

AGENTS.md 不是银弹，它不能替代代码审查、CI 或团队内部规范。但它是当前成本最低、最可控的 Agent 约束手段。把隐性知识显性化，把每次会话的重复解释变成一次性维护，是每个使用 OpenClaw 的工作空间都应该优先做的事。先写起来，再随着踩坑迭代，比追求一份完美文档更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9604fcc715bbc504.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/40ed8ebe1e496b7c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/38b9e5b66fa98e4d.png)

