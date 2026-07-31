---
title: 在 OpenClaw 中用 AGENTS.md 给 AI 代理一份真正可用的“入职手册”
feedId: 31089
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景：上下文缺失比模型能力更致命

在 OpenClaw 中编排自动化流程时，我们经常通过 `agent:` 指令将某个步骤交给 LLM 来完成，比如“根据最新数据生成日报”“重构 src/ 下的日期工具函数”“检查 docker-compose 配置的安全性”。模型本身能力很强，但真正决定输出质量的，往往不是模型规模，而是**它是否清楚自己在什么样的空间里工作**。

默认情况下，Agent 启动时只能拿到你当前 `prompt` 里塞进去的信息，以及 OpenClaw 预先注入的少量系统提示。它对项目结构、技术栈、构建命令、代码规范、团队约定一无所知。这就像让一个新同事在不看 README、不跑项目、不配环境的前提下，直接开始改代码——结果必然是一堆幻觉路径、错误的包名、不符合 CI 规范的提交。

## 问题：无界的自由等于无尽的排查

我们踩过的典型坑包括：

- Agent 自作主张在 Node.js 项目里使用 Python 依赖（因为模型“觉得更容易”）
- 生成的 Dockerfile 使用了不存在的基础镜像标签，因为不知道内部镜像仓库地址
- 重构日志模块时，没有使用项目已有的 `logger` 封装，而是重新引入了一个三方库
- 修改测试文件后，却不知道运行测试的准确命令（`pnpm test -- --coverage` vs `npm run test:ci`），导致 CI 失败

这些问题看似低级，但根因相同：Agent 缺乏一份**工作空间使用手册**。给它挂载 MCP 工具（如 filesystem、shell）可以让它“看到”文件，却不能告诉它“应该如何看待这些文件”。

## AGENTS.md：一份结构化的工作空间说明书

OpenClaw 的设计者显然意识到这个痛点。从 v0.6 开始，OpenClaw 在启动每一个 Agent 任务时，会**自动搜索工作区根目录的 `AGENTS.md` 文件**，并将其内容作为上下文的一部分注入到 system prompt 中（优先级高于默认系统提示，但低于任务级 prompt）。如果工作区是一个 monorepo，OpenClaw 会沿着目录层级向上查找最近的一个 `AGENTS.md`，同时在子目录任务中合并父级文档。

这意味着，你可以像写 README 或 CONTRIBUTING.md 一样，用 Markdown 给 AI 写一份专属手册。

## 实践：一份够用的 AGENTS.md 必须包含什么

我们经过多次迭代，最终沉淀了一个实用模板，核心分这几个块：

### 1. 项目的一句话描述与定位
```
# AGENTS.md — 为 AI 代理提供的工作空间上下文
本项目是 AcmeOps 的日志聚合控制面，基于 Go 1.22 + React 18 开发。
```
不要写作文，给出最核心的事实。

### 2. 目录速查表
提供一个轻量级树形图，并标注关键入口。避免整个 `tree` 输出，只保留 Agent 可能接触到的路径：
```
├── backend/         # Go 服务，入口 cmd/server
├── frontend/        # React 18 + Vite，所有组件在 components/
├── deploy/          # Helm chart 和 k8s 配置，不得随意修改
├── scripts/         # 构建、发布脚本
└── docs/            # 架构决策记录 (ADR)
```

### 3. 关键命令清单
不要列长列表，只给出**会被 Agent 调用的命令**：
```
## 命令
- backend 构建:   cd backend && go build ./cmd/server
- backend 单测:   cd backend && go test ./... -count=1
- frontend 开发:  cd frontend && pnpm dev
- 全栈启动:       docker-compose up -d
- CI 验证:        make ci
```

### 4. 技术约束与环境
告诉 Agent 它不能做什么，以及必须遵守的限制：
```
## 约束
- 不得引入新的第三方依赖，除非在 deps.md 中明确批准
- 所有日志必须使用 backend/pkg/log 封装，禁止直接使用 logrus/zap
- 镜像构建统一使用 hub.infra.internal/ 前缀
- 数据库 Migration 文件只能通过 scripts/migrate.sh 生成
```

### 5. 可用的 MCP 工具与边界
如果 OpenClaw 注册了 MCP 服务器（如 `playwright`、`filesystem`、`git`），在这里声明哪些可以用，以及使用规范：
```
## MCP 工具
- 允许: filesystem(read+write 仅限 /tmp/mcp), git, shell(非交互命令)
- 禁止: 执行 `rm -rf`, `kubectl delete`, 修改 /etc 下文件
- playwright 仅用于前端回归截图，不得用于爬取外部站点
```

### 6. 常见任务速览
把重复性最高的任务写成小抄，让 Agent 直接复用路径：
```
## 常见任务
- 新增 API: 在 backend/api/ 下添加 handler，在 backend/internal/router.go 注册
- 修改前端页面: 在 frontend/src/pages 创建新路由
- 生成测试数据: 运行 `scripts/seed_test_db.sh`
```

## 将 AGENTS.md 接入 OpenClaw 工作流

最简单的用法是：在项目根目录创建好 `AGENTS.md`，然后在 `claw.yaml` 的任务定义中不需要任何额外配置——OpenClaw 会自动发现并注入。你也可以显式指定路径或覆盖：

```yaml
agent:
  name: code-reviewer
  context:
    include:
      - AGENTS.md
      - .claw/extra_context.md
```

如果你有多个子项目，可以在每个子目录放置更精细的 `AGENTS.md`，父级文件会作为兜底上下文合并。注意合并不是简单的拼接，OpenClaw 会去重并保持指令优先级（子目录覆盖父级对应 section）。

如果你正在用 OpenClaw 的 `mcp` 插件系统，可以在 AGENTS.md 中直接引用 MCP 工具的提示词模板，这样 Agent 在调用时会更顺从：
```
## MCP 提示词模板
- 截图对比: "使用 playwright mcp 截图当前页面，并对比 v1.2.0 基线"
- 安全扫描: "调用 trivy mcp 扫描 backend/Dockerfile，只输出 HIGH/CRITICAL"
```

## 踩坑实录：不要把它写成又长又臭的 README

1. **信息过载 vs 信息不足**  
   最早的 AGENTS.md 我们把整个 README 加技术文档塞了进去，导致 Agent 上下文窗口爆满，处理速度骤降，关键指令被稀释。精简原则：每段话都问自己“Agent 需要这个才能做对任务吗？”不需要的就砍掉。

2. **指令冲突**  
   我们曾在一份 AGENTS.md 里同时写了“优先使用 async/await”和另一个 section 的“保持回调风格以兼容 Node 14”。Agent 陷入两难，最终含混输出。解决方案是建立一个 `## 第一优先级` 的条款，其他 section 遇到冲突时以上面为准。

3. **不同 Agent 共用一份手册**  
   一个负责代码生成的 Agent 和一个负责运维检查的 Agent 并不需要同一份手册。后来我们通过 `include` 拆分：`AGENTS.base.md` 放公共内容，`AGENTS.coding.md`、`AGENTS.ops.md` 按需引入。

4. **文档腐烂**  
   AGENTS.md 需要和代码一起维护。CI 中增加一个 check，用 `scripts/check_agents_md.sh` 校验是否存在以及关键命令是否仍然有效，否则任由它过期，Agent 会被错误引导。

5. **不应包含机密信息**  
   AGENTS.md 会被注入到远程模型的 prompt 中（除非用本地模型），不要把 API key、数据库密码、内部 host 细节写进去。敏感值应通过环境变量或 secrets 插件传递。

## 可复用的配方

- **模板化**：为团队提供 `AGENTS.md` starter，要求每个仓库初始化时填写，作为“Agent 就绪”认证的一部分。
- **分层引用**：利用 `include` 机制，把通用规范（如日志库使用、错误处理模式）抽成公共仓库的片段，再由下游项目导入。
- **和 MCP 联动**：在 AGENTS.md 中声明哪些 MCP 工具是“首选用途”，哪些是“逃生舱”（仅在特定条件下开启），让 Agent 行为更可预测。
- **用自然语言写条件规则**：Agent 理解 Markdown 的小标题和列表，用“如果是生产环境数据库，禁止任何写操作”这样直白的句式，比 YAML 逻辑更贴近 LLM 的认知方式。

## 总结

`AGENTS.md` 本质上是一份**AI 代理的接口契约**。它不增加多少维护成本，却能让 OpenClaw 中的自动化任务从“需要时刻盯着的半成品”变成“真正可以交给 AI 运行的步骤”。我们内部经历了一个从“任务 prompt 里拼命塞上下文”到“把约定写成文件，prompt 只描述可变内容”的转变，这个简化的动作实际是最有效的工程化改进。

如果你已经在用 OpenClaw 搭自动化管线，却总感觉 Agent 在犯低级错误，试一下在项目根目录放一份 80 行的 AGENTS.md，你会看到效果立竿见影。

---

