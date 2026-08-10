---
title: AGENTS.md：OpenClaw 工作空间的 AI 使用手册
feedId: 32350
source: 综合讨论
publishedAt: 2026-08-10
---

# AGENTS.md：OpenClaw 工作空间的 AI 使用手册

## 背景：当 AI 看不懂你的项目

OpenClaw 的 Agent 模式可以让 AI 直接读写文件、执行终端命令、调用 MCP 工具，在代码生成、重构和自动化场景中效率极高。但多数人很快会遇到同一个瓶颈：**AI 缺少对工作空间的持久记忆**。每次对话开始，它都会带着一套通用行为模式进入项目，不了解你的目录习惯、lint 规则、Git 提交规范，甚至不知道该用 pnpm 还是 yarn。于是第一轮生成会被大量纠正，第二次会话又要重来一遍。

我们需要的不是在每次对话时手工贴一段“项目说明”，而是一个能被 OpenClaw 自动加载的**工作空间级使用手册**。这正是 AGENTS.md 的设计意图。

## 问题：Agent 的上下文真空

默认情况下，OpenClaw Agent 能看到的只有当前文件、少数相邻文件和系统 prompt。它会根据这些有限信息做推断，结果就是：

- 新文件放错目录（组件写在 `src/components` 还是 `app/_components`？）
- 命令用错包管理器（`npm run` 而不是项目约定的 `pnpm`）
- 提交信息格式不符合 `commitlint` 规则
- 不知道项目里已经封装了自定义 MCP server，仍然用笨办法调 API

这些问题非常微妙——单独看都不致命，但会持续消耗你的修正时间，也让 Agent 从“自动”退化成“半自动”。

## 做法：给工作空间写一份 AGENTS.md

OpenClaw 规定：如果工作空间根目录存在 `AGENTS.md`，**启动 Agent 会话时会将文件内容作为系统级上下文自动注入**。这个机制本质上就是给 AI 提供一份“新员工入职手册”。

### 1. 创建 AGENTS.md 模板

放置在工作空间根目录，建议结构化，可读性优先于花哨语法。下面是一个精简可用的模板骨架：

```markdown
# AGENTS.md for [Project Name]

## 项目概览
- 技术栈：Next.js 14 (App Router), TypeScript, TailwindCSS
- 运行时：Node 20，包管理器 pnpm
- 数据库：PostgreSQL，通过 Prisma 访问
- 重要说明：本项目使用 App Router，不要生成 Pages Router 代码。

## 目录约定
- 页面组件：`app/(routes)/*/page.tsx`
- 可复用 UI：`components/ui/`
- 服务层：`lib/services/`
- 工具函数：`lib/utils/`
- 测试文件：`__tests__/` 与源文件同目录

## 代码风格
- 组件使用 `function` 声明，不用箭头函数
- 优先使用 `const` 导出，避免 default export
- CSS 仅用 Tailwind 原子类，不创建 `.module.css` 文件
- 接口命名使用 `I` 前缀（如 `IUserProps`）——这是团队遗留规范

## Git 提交
- 使用 conventional commits：`type(scope): short description`
- 允许的 type：feat, fix, chore, docs, refactor, test
- scope 用中文模块名，例如 `fix(订单): 修复价格计算异常`

## 常用命令
- 开发：`pnpm dev`
- 构建：`pnpm build`
- 测试：`pnpm test -- --watchAll=false`
- 类型检查：`pnpm type-check`
- Lint：`pnpm lint`

## MCP 工具指南
- 内部 API 文档查询：使用 `internal-docs` MCP server，工具名 `search_docs`
- 数据库 Schema 变更：调 `prisma-mcp` 中的 `validate_schema` 后再生成迁移

## 注意事项
- 避免引入新的依赖，除非明确说明
- 错误处理优先使用自定义 `AppError` 类
- 所有 API route 必须调用 `withAuth` 中间件
```

这份文件的核心是**把容易让 AI 猜错的信息固定下来**。长度控制在 1–2 KB 比较合适，太大反而稀释关键信息。

### 2. 自动加载与验证

将文件放在根目录后，下次启动 OpenClaw Agent 时，对话初始阶段就能看到 AGENTS.md 的内容。可以通过一个简单实验验证：向 Agent 提问“本项目使用什么包管理器？”如果它能正确回答 `pnpm`，说明加载成功。

如果发现没有生效，检查：
- 文件名大小写：必须是 `AGENTS.md`（全大写）
- 是否在 OpenClaw 工作区设置中关闭了“自动注入规则”选项
- 文件编码是否为 UTF-8

### 3. 与团队协作

把 AGENTS.md 纳入 Git 版本控制。项目初始化时，由技术负责人编写第一版，后续任何影响 AI 行为的约定变更（比如引入新的 lint 规则、调整目录结构）都必须同步更新该文件。这样整个团队在使用 OpenClaw 时，Agent 都能拿到一致的工作空间知识。

## 踩坑点

**1. 文件越长，注意力越散**
一开始我把 AGENTS.md 写成了 8 KB 的项目文档，Agent 反而容易忽视其中的关键条目。后来精简到 1300 字左右，命中率明显提升。建议用短句、列表和明确的否定句（如“不要生成……”）。

**2. 动态内容不适合写死**
不要在这里放 sprint 编号、当前修复的 issue 链接等动态信息。这些应该用临时 prompt 或会话上下文传入。AGENTS.md 保持相对稳定。

**3. 敏感信息绝对不能出现**
即使仓库是私有，也不要在 AGENTS.md 里放 API 密钥、内网地址等。OpenClaw 可能会将内容发送到云端模型服务，这一点必须谨慎。

**4. 和 `.openclawignore` 配合**
如果你的 MCP 工具会读取文件，而 AGENTS.md 里又引用了某些私有目录，记得在 `.openclawignore` 中排除，避免 Agent 试图访问不该碰的文件。

## 可复用建议

- **模板化**：不同项目可以共用同一套 AGENTS.md 骨架，只修改加粗部分。我们团队内部维护了一个 `agent-manual-template.md`，新建仓库时直接复制。
- **版本标记**：在文件末尾加一行 `<!-- ver:1.2 -->`，方便快速检查是否过期。
- **关联 lint 配置**：如果 AGENTS.md 里声明了代码风格，尽量让它和 ESLint/Prettier 配置一致。Agent 会同时读两者，不一致时可能产生奇怪输出。
- **做减法试验**：如果不确定某条规则是否必要，可以注释掉运行一周，观察 Agent 质量变化。这比盲目增补更靠谱。

## 总结

AGENTS.md 本质上是**对工作空间上下文的工程化管理**，而不是玄学提示词。它解决的不是 Agent 能力上限的问题，而是能力浪费的问题——当你不再需要反复矫正 AI 的基础行为，注意力才能放到真正复杂的任务上。

如果你的 OpenClaw Agent 仍在新会话里用错包管理器、放错文件位置，不妨现在就在项目根目录新建一个 `AGENTS.md`，把第一条规则写进去。花 15 分钟把隐性的项目知识显性化，也许是本季度 ROI 最高的工程投资。

---

