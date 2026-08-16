---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 33478
source: 综合讨论
publishedAt: 2026-08-17
---

最近在 OpenClaw 里跑本地自动化，发现一个规律：每次新开 session，模型对项目约定一无所知。明明同一个仓库，上个会话还知道用 pnpm、测试命令是 `pnpm test`，下个会话就开始乱用 npm，甚至把构建产物写进源码目录。

与其每次在 prompt 里贴一段“项目规则”，不如让 OpenClaw 自动加载 AGENTS.md。

## 背景

OpenClaw 允许 Agent 在工作空间内执行 shell、读写文件、调用 MCP 工具。但模型本身对目录结构、包管理器、构建命令、测试入口没有稳定记忆。会话一切，上下文就丢。

AGENTS.md 的作用，就是把这些容易跑偏的工程约束固化成工作空间内的“使用手册”。它随着项目走，而不是随着某次会话走。

## 问题

没有 AGENTS.md 时，我遇到几个反复出现的情况：

- 同一个项目，Agent 在不同 session 里随机切换 npm / yarn / pnpm，导致锁文件互相污染。
- 生成物写到 `src/` 下，而不是约定的 `output/`。
- 服务启动目录错误，例如在子目录里找 `.env`，然后直接报错。
- 数据库操作绕过 MCP 工具，直接尝试执行 SQL 文件。

这些问题不算大，但每次都要口头纠正，非常消耗耐心。更糟的是，长任务跑到一半才发现路径或命令错了，前功尽弃。

## 做法 / 步骤

我把 AGENTS.md 放在工作空间根目录，和 `.git` 同级。OpenClaw 默认加载当前工作区的 AGENTS.md，如果 cwd 是子目录，需要确认是否会向上查找。我的习惯是固定工作区路径，不让 Agent 在临时目录里乱跑。

AGENTS.md 我一般只写五段，控制在 60 行以内：

1. **Project map**  
   关键目录及职责，例如：
   - `src/`：源码，只允许修改业务代码。
   - `output/`：所有生成物必须写到这里。
   - `scripts/`：可执行脚本，不随意改。
   - `fixtures/`：测试数据，禁止写入真实数据。

2. **Commands**  
   写“必须”，不写“推荐”：
   - 安装依赖：`pnpm install --frozen-lockfile`
   - 测试：`pnpm test`
   - 构建：`pnpm build`
   - 启动：`pnpm dev`

3. **Runtime & env**  
   要求 Node 20+，需要 `.env.local`，但禁止读取或修改生产 `.env`。这里不写任何真实密钥。

4. **Constraints**  
   硬性禁止事项：
   - 禁止删除或修改 lock 文件。
   - 禁止在 `output/` 外生成产物。
   - 禁止修改 `migrations/` 下的历史迁移文件。
   - 数据库变更只能通过 `mcp__db__migrate`，不允许直接执行 SQL。

5. **Done criteria**  
   任务完成前必须满足：
   - 生成的文件已放在 `output/`。
   - 已跑过 `pnpm test` 且通过。
   - 已执行 `scripts/check-workspace.sh`，没有报错。
   - 在回复里给出可复现的命令序列。

写完 AGENTS.md 后，可以新开一个 session 测试自动加载。直接问 Agent：“当前工作空间的包管理器和测试命令是什么？”如果回答能和 AGENTS.md 对上，说明加载生效。

另外，我加了一个 gate 脚本 `scripts/check-workspace.sh`，检查 lock 文件是否被改、`output/` 外是否有新文件、是否有未格式化的代码。AGENTS.md 里明确写：完成前必须运行这个脚本，失败则不要结束任务。这样比单纯依赖模型自觉可靠得多。

## 踩坑点

1. **AGENTS.md 不是越长越好**  
   超过 150 行后，模型容易忽略靠后的规则，或者把示例当成可执行内容。详细说明放到 `docs/`，AGENTS.md 只留指针。

2. **“推荐”和“必须”差异很大**  
   我早期写“建议使用 pnpm”，结果 Agent 遇到 pnpm 报错时直接切到 npm，生成了 `package-lock.json`。改成“只允许 pnpm，禁止生成 `package-lock.json`”后才稳定。

3. **不要把机密写进去**  
   AGENTS.md 会被拼进上下文，可能通过错误日志或长会话泄漏。密钥、内网地址、个人 token 一律不写。

4. **AGENTS.md 会过时**  
   迁移目录或升级工具链后，旧规则会误导 Agent。我在 CI 里加了一个简单 lint：检查 AGENTS.md 提到的路径是否存在，超过 7 天未验证就提示更新。

5. **加载范围要确认**  
   如果 cwd 是子目录，OpenClaw 可能不会向上查找 AGENTS.md。临时目录跑任务时，Agent 可能读到错误上下文。固定工作区路径可以避免这个问题。

## 可复用建议

- 把 AGENTS.md 当作“工作空间宪法”，只写高频、容易跑偏的内容。具体操作步骤丢给脚本或 MCP 工具描述。
- 和 MCP 配合：如果某个能力已经通过 MCP server 暴露，AGENTS.md 里明确写“只能走 MCP 工具，不允许绕过”。
- 让规则可执行：把“完成前运行检查脚本”写成硬性 gate，而不是软性建议。
- 给规则加版本号。变更 AGENTS.md 后，commit message 写清楚，避免旧 session 用旧规则继续跑。
- 保留一个最小模板，新仓库直接复制。模板控制在 50 行以内，按需增删。

## 总结

AGENTS.md 的价值不是“教 AI 做人”，而是把反复交代的工程约束固化成版本化、可检查的上下文。它解决的是 Agent 在长任务中漂移的问题。

别追求大而全。先写五段，跑两周，把每次需要口头纠正的内容沉淀进去，再删掉已经不痛不痒的规则。对 OpenClaw 用户来说，AGENTS.md 比 prompt 里的长篇大论更可靠，因为它随项目走，而不是随会话走。

---

