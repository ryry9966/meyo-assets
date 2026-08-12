---
title: 让 AI 帮你提 commit、切分支：一个可落地的 Git 自动化实践
feedId: 32750
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：提交信息和分支管理，为什么总是拖后腿

在多人协作的工程里，规范化的 commit message 和分支命名往往是“重要但不紧急”的事。开发者赶进度时，`fix bug`、`update` 这样的提交信息随处可见，分支名也是临时的 `test-123`，事后追溯变更历史变成考古现场。

另一方面，类似的重复性操作——查看 diff、概括变更、切出符合规范的分支、推送到远端、创建 PR——这些步骤本身逻辑固定、模式清晰，天然适合交给自动化。Git 的 CLI 已经足够强大，但缺乏一个能“读得懂变更、做得出决定”的调度层。而现在的 AI Agent 正好能填补这个空缺，把“看懂代码做了什么”和“执行 git 命令”串成一个可审计的工作流。

这篇文章面向已经在使用 OpenClaw 或 MCP 做自动化的工程师，分享一套在真实仓库里实践过的方案：让 AI 助手自动生成 Conventional Commits 格式的提交信息，管理分支生命周期，同时保留必要的人工卡点，避免自动化变成新的事故源。

## 问题拆解：我们要解决哪几件事

一个实用的 Git 自动化流程需要覆盖：

1. 能够理解暂存区或工作区的变更，生成结构化的 commit message（如 `<type>(<scope>): <subject>`）
2. 按团队的命名规则自动创建分支（如 `feat/xxx`、`fix/xxx`）
3. 在推送前对信息做校验，避免明显错误
4. 推送后可选地创建 PR，并把 AI 分析出的变更摘要写入 PR 描述

另外，整个过程不能变成完全黑盒的自动提交。仓库的主分支保护、CI 卡点、人工 Review 基座不能丢。

## 做法：用 MCP Git Server 把仓库暴露给 Agent

这里的核心工具是一个 MCP Git Server（社区已有开源实现），它的作用是让大语言模型可以通过标准化的 MCP 工具接口，执行 `git status`、`git diff`、`git add`、`git commit`、`git branch`、`git push` 等命令。结合 OpenClaw 的 Agent 能力，我们可以用自然语言定义任务，模型会自驱地调用这些工具完成操作。

**步骤一：部署 MCP Git Server 并接入 OpenClaw**

我选择在一个容器里运行 git server，挂载宿主机的代码目录，只读挂载 `.gitconfig` 以便使用开发者的签名信息。然后通过环境变量限制可以执行的操作，例如禁止 `push --force`、禁止操作 `main` 分支，只允许 push 到 `feat/*`、`fix/*` 等命名空间。

将服务注册到 OpenClaw 的 MCP 配置中，测试基本的读操作：
```
[git-server]
command = "node"
args = ["/path/to/git-mcp-server/dist/index.js"]
env = { "GIT_ROOT": "/workspace", "ALLOWED_BRANCH_PATTERN": "feat/*,fix/*" }
```

Agent 就能通过工具调用看到仓库状态和 diff。

**步骤二：定义自动化任务**

我常用的 Prompt 模板大致如下：

> 你是我的 Git 助手。现在暂存区有一组变更，请做以下事情：
> 1. 用 `git diff --cached` 查看改动，理解变更目的。
> 2. 生成一个符合 Conventional Commits 的提交信息，格式：`type(scope): subject`，scope 尽量具体，subject 用英文概括，不超过 72 字符。如有破坏性变更，追加 `BREAKING CHANGE:` 页脚。
> 3. 根据 type 和 scope 生成分支名，规则：`$type/$scope-YYMMDD`。例如 `feat/api-240801`。
> 4. 先切到新分支，创建提交，然后推送。
> 5. 如果仓库启用了 PR 模板，用 `gh pr create` 创建 PR，标题为 commit subject，正文是一段包含变更要点、影响范围和测试建议的摘要。

将这个 Prompt 绑定为一个 OpenClaw Skill，开发者暂存文件后说一句“帮我提个 PR”，Agent 就会串行完成以上动作。

**步骤三：加入人工确认 Gate**

全自动提交风险太大，我人为增加了一个“暂停-确认”环节：在 Agent 实际执行 `git commit` 和 `git push` 前，要求它先把提交信息和分支名输出给用户，并等待明确的“确认”指令。这个可以通过在 Skill 中插入类似 `ask_user` 的工具实现。一次典型的交互如下：

```
Agent: 我准备使用以下信息创建提交和分支：
  Commit: feat(order): add date range filter to list endpoint
  Branch: feat/order-240801
是否继续？
用户: 确认
Agent: 已创建提交并推送至 origin/feat/order-240801。PR 已创建：#452。
```

这样一来，AI 只负责生成建议和脚本执行，最终决定权还在人手里，质量风险可控。

## 踩坑点与工程化经验

在实际跑了一段时间后，有几个点值得注意：

**1. Diff 尺寸与上下文窗口**

大范围重构时 `git diff` 输出可能几千行，直接塞给模型会超 token 或产生幻觉。我的做法是对 diff 做预处理：只取每个文件的统计行和关键函数名，如果变更过于分散，就要求用户分批暂存。也可以在 MCP 层做一个简单的 diff 摘要工具，模型先看摘要，再按需读取具体文件 diff。

**2. 生成的 scope 有时过于宽泛**

AI 喜欢用 `core`、`general` 等模糊 scope，但这种 scope 对追溯帮助不大。这里通过 Prompt 约束统一使用模块名（如目录名），并在代码仓维护一个 `.scope_map` 文件，Agent 在生成前先读取，从实际变更文件路径推导 scope。这样 commit 信息能保持模块粒度的对齐。

**3. 分支名冲突与重提交**

如果分支名已存在，Agent 默认操作会直接失败。在 Skill 中加入异常处理：当 `git branch` 显示分支存在时，追加时间戳后缀重试一次，或者提醒用户手动处理。不建议让 AI 自动对已有分支进行变基或合并，那超出自动化舒适区。

**4. 权限边界必须硬限制**

这是最容易出事的地方。MCP Git Server 的“允许操作”白名单必须在服务端强制约束，而不是依赖 Prompt 告诉 AI “不要 force push”。我的做法是：在 git server 的包装层对危险命令进行字符串级别拦截，返回“命令不允许”错误；同时在 GitLab/GitHub 的 branch protection 上也设置保护，双重保险。

## 可复用建议

如果你要在自己的团队里推广这套流程，几条务实建议：

- **小步启动**：先只做 commit message 生成 + 人工确认，分支和 PR 创建先不动。让团队成员习惯“AI 帮你写提交信息”的交互模式，建立信任后再逐步纳入分支和 PR。
- **坚持人的审批**：在质量相关的步骤（最终提交信息、分支名、是否 push）上保留人工确认，不要把自动化做成无人值守的“提交机器人”。
- **把规范模板化**：commit 规范、分支命名规则、PR 描述模板全部写成可被 AI 读取的 Markdown 文件，放入仓库 `docs/` 或 `.github/`，Agent 在生成前强制阅读。这样所有开发者都在同一个规范约束下提交。
- **监控与回溯**：在 Agent 日志中记录每次工具调用和模型输出，出现异常时能够快速定位是模型理解偏差还是工具执行错误。
- **与现有 CI 集成**：AI 生成的提交同样需要通过 lint-commit、eslint、测试流水线，不搞特殊通道。自动化只是前置环节，质量闸门不变。

## 总结

Git 自动化不是要取代开发者的提交决策，而是把“翻译”和“搬运”这两层体力活交给 AI：从 diff 翻译成规范化的提交信息，把分支规则、PR 模板搬运成型。这个过程中，Agent 的价值在于理解变更和遵守规则，而人的价值在于判断和确认。

在 OpenClaw + MCP 的框架下，这套自动化的搭建成本已经大幅降低——只需要部署一个受控的 MCP Git Server，配好权限模型和 Prompt 模板，就可以把 AI 接入团队的 git 工作流。关键是控制好“什么该自动、什么必须人批”的边界，让自动化成为可靠的协作成员，而不是一个提完 `feat(core): update` 就甩锅走人的 AI 同事。

---

