---
title: AGENTS.md：写给 AI 的工作空间使用手册——让 OpenClaw 不再“猜”你的项目
feedId: 32092
source: 综合讨论
publishedAt: 2026-08-08
---

## 一、背景：每个 AI Agent 都在“盲猜”你的工程约定

在 OpenClaw / Agent / MCP / 插件式的自动化实践中，我们通常会花大量精力构建工具链、编排流程、调优 prompt。但有一个极易被低估的问题：**Agent 对项目上下文的理解完全依赖你喂进去的散装信息**。

无论是通过 OpenClaw 触发一次代码重构，还是让一个 MCP 工具自动提交 PR，AI 真正能“看见”的只有 system prompt 中的通用规则和当前会话注入的片段。如果你没告诉它：

- 这个 monorepo 的包管理器是 pnpm 还是 yarn；
- 测试命令是 `vitest` 而不是 `jest`；
- 哪个目录是自动生成的，绝对不能直接修改；
- 配置文件里哪些字段只读、哪些需要锁版本……

那么 Agent 做出“看上去合理但完全错误”的操作只是时间问题。  

在 Obsidian / Copilot / Cursor 生态里，`.cursorrules`、`COPILOT.md` 已经成为一个廉价的“项目说明书”。但在 OpenClaw 的自动化工作流里，我们缺一个同样轻量、可注入、可版本化的标准文件。这就是 **AGENTS.md** 的用武之地。

## 二、问题：散装上下文的三个致命缺陷

1. **高维护成本**  
   每次新建 OpenClaw skill 或 workflow，都要在 prompt 里复制一遍项目结构、依赖版本、路径别名。一个改动漏改一处，Agent 就开始编造文件路径。

2. **行为不一致**  
   同一个 Agent，通过不同入口（MCP 工具调用 vs 插件对话）执行相似任务，得到的结果却可能因为上下文不一样而大相径庭。

3. **难以沉淀团队知识**  
   到底是先用 `npm run generate` 再改代码，还是直接手写？这些约定只存在于老员工的 Notion 文档里，Agent 永远学不会，新人也要反复踩坑。

这些问题本质不是模型能力不够，而是**工程上下文没有结构化成机器可读的固定规范**。

## 三、做法：为 OpenClaw 工作空间创建 AGENTS.md

### 1. 文件定位
在项目根目录创建 `AGENTS.md`。它和 `README.md` 的区别在于：
- README 是给人看的，侧重业务背景、开发指南；
- AGENTS.md 是**给 AI Agent 看的**，侧重可执行指令、硬约束、路径映射、禁止行为。  

如果项目很大，可以进一步拆成 `AGENTS.md`（总则）+ `docs/agents/` 下的细分页面，然后用引用链接组合。

### 2. 最小可用内容模板
一个生产可用的 AGENTS.md 至少包含这几块：

```markdown
# AGENTS.md

## 项目技术栈
- Runtime: Node.js 20
- 包管理器: pnpm (必须，禁止使用 npm / yarn)
- 测试: vitest
- 构建: tsup
- Lint / Format: eslint + prettier

## 目录约定
- `src/`：源代码
- `src/generated/`：**自动生成，禁止 AI 直接编辑**
- `tests/`：集成测试，文件名后缀 `.test.ts`
- `.openclaw/`：OpenClaw 工作流及 MCP 配置，需保持与 AGENTS.md 一致

## 常用命令
- 启动开发：`pnpm dev`
- 运行测试：`pnpm test -- --run`
- 类型检查：`pnpm typecheck`

## 代码规范
- 严禁使用 `any`，除非已有 JSDoc 说明理由
- 新增接口必须同时导出类型到 `src/types/public.ts`
- 修改配置文件后须运行 `pnpm config-sync`

## 安全边界
- 禁止读取或输出 `.env`、`.env.local` 中的值
- 禁止修改 `.gitignore` 中列出的 untracked 文件
- 任何涉及 `exec` 的操作需显式确认

## AI 行为约束
- 生成文件前需先列出将要创建/修改的文件清单
- 若不确定某项约定，优先执行 `pnpm run guide` 查阅项目内置帮助
```

### 3. 在 OpenClaw 中注入 AGENTS.md
目前有两种低成本做法，不需要改动框架源码：

- **通过 MCP 工具自动读取**  
  创建一个简单的 MCP resource（或利用 `filesystem` MCP 服务器），让 Agent 在对话开始或每次任务执行前读取 `<project_root>/AGENTS.md`。可以在 OpenClaw 的 `workspace` 配置里绑定一个预置的 `read_project_context` tool。

- **利用 OpenClaw 的 global prompt 注入**  
  在 `openclaw.yaml` 的 `global.prompt` 或对应 workspace 的 `systemPrompt` 中，用 `${file:AGENTS.md}` 占位符（如果框架支持）或直接用脚本拼接。更稳定的方法是在 CI 中生成一个 `AGENTS.txt` 供框架直接消费，避免运行时解析 Markdown。

**实操示例：**  
在 OpenClaw 的 workspace 配置中加入：
```yaml
workspace:
  prompts:
    - id: project-agent-md
      type: file
      path: ./AGENTS.md
      injectAt: beginning
```
这样每次 Agent 启动时都会把整个 AGENTS.md 作为 system prompt 的最前序内容，确保后续所有操作都受其约束。

## 四、踩坑记录：AGENTS.md 不起作用的六个原因

1. **文件过长，token 被隐性裁剪**  
   写了几百行，结果模型实际只读到前 2000 字符。解决方法：把详细说明下放到独立文件，AGENTS.md 只保留索引和最高频的硬约束。

2. **包含绝对路径**  
   `/Users/me/projects/foo` 这种路径在 CI / Docker 里直接失效。AGENTS.md 必须以项目根为基准的相对路径，或使用 `$PROJECT_ROOT` 这类变量。

3. **与 System Prompt 冲突**  
   如果 OpenClaw 的全局 prompt 已经写了“代码风格按 Google Style”，而 AGENTS.md 却指定了 Airbnb Style，Agent 会摇摆或无视。需在团队内约定 AGENTS.md 为唯一事实来源，全局 prompt 只做通用人格设定。

4. **没有版本控制感知**  
   AGENTS.md 修改后，旧版 checkpoint 仍在 Agent 的上下文缓存里。建议在文件头加入 `version: 2.3`，并定期清理对话或设计一个 `/reload-context` 指令。

5. **信息过时**  
   人们修改了脚本命令却不更新 AGENTS.md，Agent 开始频繁执行错的命令。解决办法：在 CI 里加一步校验，如果 `pnpm test` 命令与 AGENTS.md 中记录不一致则 block merge。

6. **隐私与安全**  
   避免在 AGENTS.md 中直接写任何密钥、内网 IP、内部域名，因为该文件可能随日志或被 Agent 引用到外部工具。敏感信息用环境变量引用，并由 MCP 工具在运行时注入。

## 五、可复用建议：把 AGENTS.md 变成工程能力而非文档

- **模板化**：为团队维护一份 `AGENTS.template.md`，新项目直接复制并填写项目特有部分。
- **条件加载**：根据 OpenClaw workspace 的类型（前端、后端、数据管道）加载不同片段，例如 workspace 为 `fe` 时自动注入 `AGENTS.frontend.md`。
- **同步工具**：写一个小脚本，从 `package.json` 的 scripts 字段自动生成 AGENTS.md 中的命令部分，减少人工维护。
- **可观测性**：在 Agent 操作日志中记录“本次任务使用的 AGENTS.md 版本”，方便回溯为什么 Agent 执行了某个步骤。

## 六、总结

AGENTS.md 本质上是一份**为 AI 编写的项目工作空间使用手册**。它不是银弹，但可以低成本地消灭大量“明明之前告诉过它”的错误。在 OpenClaw 这类以 Agent 编排为核心的框架里，确保所有自动化行为都在一份受控、可追溯、可更新的规则下运行，比不断优化单个 prompt 要重要得多。

下次当你准备把新的业务逻辑交给 OpenClaw Agent 去实现时，先花 10 分钟更新一下 `AGENTS.md`——这 10 分钟会从后续数十次“为什么会这样”的排查中补偿回来。

---

