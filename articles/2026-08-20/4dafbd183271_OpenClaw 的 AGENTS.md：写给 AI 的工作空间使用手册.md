---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 33906
source: 综合讨论
publishedAt: 2026-08-20
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景：Agent 很聪明，但不知道你的项目长什么样

最近在 OpenClaw 工作空间里频繁切换不同仓库，发现一个反复出现的问题：每个项目都有自己独特的构建方式、目录约定和开发脚本，但 AI Agent 每次进来都是从零开始猜。让它跑个测试，可能先执行了 `npm test` 而不是项目实际用的 `pnpm vitest run --coverage`；让它改文档，它按通用习惯找到 `docs/`，却不知道这个仓库实际放在 `content/posts/` 下。

本质不是模型能力问题，而是**工作空间级别的上下文缺失**。MCP 能提供外部工具接入，插件能扩展运行时能力，但项目本地的结构与约定，仍然需要一种标准化方式告诉 Agent。

## 问题：`README.md` 是给人读的，不是给 Agent 读的

很多人会问：README 不是已经有说明了吗？

问题在于 README 的定位偏差：

- README 面向**人类贡献者**，包含项目宣传、徽章矩阵、演示 GIF、贡献者列表。这些内容对 Agent 来说大半是噪声。
- README 不会写"修改 `src/parser/` 下的文件后必须同步更新 `snapshots/` 下的 fixture"，这类约定散落在团队 wiki 或个人笔记里。
- Agent 的上下文窗口是稀缺资源。让它每次读 200 行带图 README，不如读 40 行精炼指令。

我需要一个约定俗成的入口文件，让 Agent 进工作空间后优先读取，而且内容只写"帮我高效工作"的信息。

## 做法：建立 AGENTS.md 作为 Agent 的默认入口

### 第一步：创建工作空间根目录的 `AGENTS.md`

位置放在仓库根目录，与 `.git/` 同级。OpenClaw 会在工作空间上下文中优先注入这个文件。

### 第二步：按"Agent 视角"选题，而不是人类读者视角

我删掉了所有营销文案、徽章、演示截图。只保留 Agent 真正需要的信息块：

```
# AGENTS.md

## Build & Test
- 包管理：pnpm，不要使用 npm/yarn
- 测试：pnpm test（实际运行 vitest run --coverage）
- 类型检查：pnpm typecheck，与测试相互独立

## Repo Layout
- src/parser/ ：语言解析器，修改后必须更新 snapshots/
- snapshots/ ：解析器输出的 golden files，由 pnpm update-snapshots 生成
- tools/ ：CLI 辅助脚本，不经由 CI，勿在正式流程中引用

## Conventions
- 所有用户可见字符串必须抽出到 locales/zh-CN.json
- API 路由的响应格式统一用 { data, error } 包裹
- 不要自行引入新的运行时依赖，除非在 proposal 中说明
```

### 第三步：让 Agent 实际去读取它

在 OpenClaw 中，AGENTS.md 不是魔法。我会在任务引导语中显式标注："Read AGENTS.md first, follow repo conventions." 对于不自动读取的 Agent 场景，这是必要的兜底。

同时把命令类信息写成 Agent 可以直接复制执行的格式，比如：

```
## Commands
- Dev server: pnpm dev --port 3000
- Single test: pnpm vitest run src/parser/__tests__/tokenizer.test.ts
```

这比写"本项目使用 pnpm 作为包管理工具"这种散文式描述更可用。

## 踩坑点

### 1. AGENTS.md 写太长，Agent 也不读

第一版我写了 200 多行，涵盖 CI 流程、CD 部署、监控告警。结果 Agent 经常忽略关键约定。后来砍到 50 行以内，每条约定必须有"违反的后果"或"对应的命令"。信息密度比覆盖面重要。

### 2. 不要只写"应该怎么做"，要写"别做什么"

负面指令对 Agent 的约束力明显更强。比如：

- ✅ "测试必须通过 `pnpm test`，不要直接调用 `vitest`，因为需要额外的 setup 环境变量"
- ❌ "请使用项目统一的测试命令"

前者写清了原因，Agent 违反的概率更低。

### 3. AGENTS.md 不会自动失效

改目录结构、换 CI 流程后，如果没更新 AGENTS.md，Agent 会拿着过时约定干活——比没有约定更危险。我把 AGENTS.md 的更新纳入了 PR checklist：**任何改变构建/测试/目录结构的提交，必须同步更新 AGENTS.md**。

## 可复用建议

- **单一入口原则**：AGENTS.md 是导航层，不承载细节。细节让 Agent 按需读指定文件。例如写"部署流程详见 `deploy/README.md`"，而不是把整个流程塞进 AGENTS.md。
- **分层管理**：单仓多项目（monorepo）时，根目录放全局约定，每个子包放自己的 AGENTS.md，结构类似：
	```
	/AGENTS.md              # 全局：包管理、monorepo 协议、变更流程
	/packages/web/AGENTS.md # web 子项目：路由约定、组件规范
	/packages/api/AGENTS.md # api 子项目：中间件顺序、错误码规范
	```
- **保持"可执行"**：写命令时用完整可复制的形式，Agent 不需要推理参数。
- **版本化**：AGENTS.md 跟随 git 走。不要放在 gitignore 里，不要写入个人偏好。

## 总结

AGENTS.md 解决的不是"让 Agent 变聪明"的问题，而是**减少 Agent 因为缺少项目常识而犯低级错误**的问题。它本质上是一份写给 AI 的工作空间使用手册，要求的是精炼、可执行、有约束力。

在一个 MCP 和插件生态越来越丰富、Agent 能力越来越强的环境里，工作空间级别的本地约定仍然是不可替代的上下文。因为外部工具可以接入，但只有你知道自己的项目有哪些"坑"。

文件很短，工程价值很长。

---

