---
title: 用 OpenClaw Agent 接管日常 Git 提交与分支管理
feedId: 31911
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在多人协作的项目里，规范的 commit message、清晰的分支命名、及时的 PR 描述是工程质量的底线。但现实是，开发者经常在赶进度时随手写一句 “fix bug” 或 “update”，分支名也变得随意，后期追溯和自动化 changelog 都成了灾难。让 AI 助手介入这些高频又重复的操作，能有效降低心智负担，且不改变既有开发流程。

OpenClaw 的 Agent + MCP 工具组合提供了一种轻量的侵入方式：你可以让 Agent 监听文件变更、自动生成符合 Conventional Commits 规范的提交信息，甚至在 TAPD/Jira 工单驱动下自动创建带编号的分支。本文记录我在一个内部项目中的落地过程。

## 问题抽象

目标是将下面这些机械动作交给 Agent：

1. 文件保存后，自动 `git diff` 获取变更内容
2. 根据 diff 和项目上下文，生成带 type/scope 的 commit message
3. 执行 `git add` 和 `git commit`（不自动 push）
4. 当从工单系统拉取到任务时，自动创建 `feature/TICKET-123-description` 风格的分支

需要解决的工程问题包括：diff 长度受限于模型上下文窗口、生成结果需要后校验、如何在本地安全地调用 Git 命令。

## 做法 / 步骤

### 1. 环境与依赖

- OpenClaw Agent（版本 ≥2.0，支持 MCP 协议）
- MCP Git Server（社区版即可，暴露 `git_diff_unstaged`、`git_commit`、`git_create_branch` 等工具）
- 一个用于文件监控的轻量脚本（我用的是 `entr` 配合 shell，也可以直接用 Agent 的轮询模式）

确保 `git` 在 Agent 运行环境中可用，且配置了用户信息。

### 2. Agent 配置

在 OpenClaw 的 Agent 配置中挂载 Git MCP 工具，并定义一个编排：

```yaml
agent:
  name: git-assistant
  tools: [mcp-git]
  triggers:
    - type: file_watcher
      paths: ["src/**/*.ts", "src/**/*.tsx"]
      exclude: ["node_modules", ".git"]
      action: auto_commit
  actions:
    auto_commit:
      steps:
        - mcp: git_diff_unstaged
          save_as: diff_output
        - prompt: commit_message_generator
          inputs: { diff: "$diff_output" }
          save_as: commit_msg
        - condition: "commit_msg not empty and pass_validation"
          then:
            - mcp: git_commit
              params:
                message: "$commit_msg"
                files: "."
```

这里的关键点是先把 `git_diff_unstaged` 的结果保存为上下文变量，再送入生成 prompt，最后条件执行提交。

### 3. 生成 commit message 的 Prompt 设计

直接用原始 diff 容易超出 token 限制，且模型容易分心。我做了两个优化：

- 若 diff 行数 > 200，则先让 Agent 用 `git diff --stat` 获取概要，再调用模型生成一条概括性 message，最后用 `--no-verify` 跳过本地 hook（因为已经在生成阶段做过规范检查）。
- 在 prompt 中显式要求输出格式，且禁止解释性文字：

```
你是一个严格的代码提交助手。
根据下面的 unified diff，生成一条符合 Conventional Commits 的 message。
只输出最终 message，不要任何额外内容。

规则：
- type: feat, fix, refactor, docs, test, chore 等
- scope 可选，用中文简短描述
- subject 用祈使句，不超过 72 字符
- 如果 diff 中同时包含功能和重构，优先取主要意图

diff:
--- a/src/auth/login.ts
+++ b/src/auth/login.ts
...
```

模型返回后，Agent 会做一次正则校验 `^(feat|fix|...)\(?.*\)?: .{1,72}`，不通过则降级为手动模式——在终端打印推荐 message 并等待确认。

### 4. 自动分支创建

另一个常见场景：从项目管理工具拉取到新任务时需要建分支。我写了一个简单的 MCP 工具桥接，接收 TAPD 的 story_id 和标题，然后调用 `git_create_branch`：

```
branch_name = f"feature/story-{story_id}-{slug(short_title)}"
```

Agent 在收到 TAPD webhook 或定时轮询到的未处理工单后，自动在本地创建分支并推送到远端，同时在工单上添加一条评论告知分支名。这里要注意 slug 生成时限制长度 ≤ 30 字符，避免过长。

## 踩坑点

1. **大 diff 导致 token 超出**：项目首次接入时整个文件重写的 diff 可能超过 8192 token，直接报错。解决方案是先用 `--stat` 降级，或只对部分文件类型启用自动提交，核心模块仍保留手动。

2. **生成 message 偶尔 “幻觉”**：模型会添加 “Closes #123” 这样的尾部，但我们项目并不用 GitHub issues，会在 commit 时注入无用信息。通过后处理正则 `^\s*Closes\s` 去掉多余 footers 解决。

3. **git 操作权限与安全**：Agent 运行在开发机本地，天然有所有 git 权限，一旦 prompt 被恶意构造可能导致危险命令。因此绝不可让 Agent 执行 force push 或删除分支等操作。我只在 MCP 工具定义中暴露了 `git_add`、`git_commit`、`git_create_branch`，且 `commit` 时不允许 `--amend`。

4. **文件监控的 “抖动”**：短时间内多次保存会触发多次 diff 和 commit，可能造成空提交。通过比对最新 commit 的 tree hash 与当前工作区的 tree hash，若相同则跳过。利用 `git diff --quiet` 判断工作区是否干净。

5. **项目配置文件中的路径兼容**：Windows 与 macOS 的路径分隔符差异，在文件监控的 exclude 模式上需要指定 POSIX 风格，我统一使用 `/`。

## 可复用建议

- **渐进式启用**：先对非核心模块启用自动提交，例如 docs、changelog、config 等变更，验证稳定后再扩展。
- **人工兜底**：永远保留一块 “建议模式”，Agent 生成后通过桌面通知或终端展示 message，用户 10 秒内无操作则自动提交，否则可手动修改。这样可以兼顾效率和安全感。
- **模板化 prompt**：将 commit 规则、scope 白名单维护在外部配置文件里，Agent 每次加载，便于不同项目复用。
- **与 CI 联动**：在 pre-receive 或 CI 流水线中加入相同的 message 规范校验，防止自动生成的 commit 绕过检查。
- **监控与回滚**：记录所有 Agent 发起的 git 操作到日志，一旦出现问题可以快速定位。

## 总结

Git 自动化并不是要替换开发者的决策，而是像一位严格的秘书，帮你记住规范、减少琐碎操作。使用 OpenClaw Agent + MCP Git 工具的方案，能在最小侵入下实现自动化的提交信息和分支管理。经过一个多月的运行，我们团队的 commit 规范性从 60% 提升到 92%，分支命名不再出现 `aaa` 或 `test`。当然，这背后需要细粒度的工具设计、合理的 token 管理以及最后一道人工校验。如果你也在被 Git 琐事困扰，不妨从这个轻量方案开始尝试。

---

