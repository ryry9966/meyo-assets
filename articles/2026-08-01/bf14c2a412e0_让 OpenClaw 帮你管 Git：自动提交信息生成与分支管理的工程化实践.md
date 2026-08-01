---
title: 让 OpenClaw 帮你管 Git：自动提交信息生成与分支管理的工程化实践
feedId: 31252
source: 综合讨论
publishedAt: 2026-08-01
---

# 让 OpenClaw 帮你管 Git：自动提交信息生成与分支管理的工程化实践

## 为什么需要 Git 自动化助手

日常开发中，重复性最高的操作不是写代码，而是围绕 Git 的机械劳动：暂存、写提交信息、切分支、同步上游、整理 PR 描述。团队一旦要求 Conventional Commits 规范，产出的信息要么是千篇一律的“fix bug”，要么是事后尴尬地 rebase 改历史。分支命名同样容易放飞自我，`feature-X`、`test-branch` 混在一起，几个月后的考古成本极高。

与其每次都手动去翻 diff 写描述，不如让 AI 助手承担这些低智力密度的环节。在 OpenClaw 的生态里，Agent 可以通过 MCP 协议直接调用 Git 命令，分析变更内容并输出符合规范的提交信息，甚至能依据任务描述自动创建标准化分支。这不是科幻，而是已经能在本地跑起来的现实工作流。

## 我们要解决的具体问题

1. 提交信息不规范：`fix`、`update`、`wip` 泛滥，自动生成 changelog 时完全不可用。
2. 分支命名混乱：临时分支、个人分支、功能分支平铺，无法追溯需求来源。
3. 合并请求描述空白：PR 标题自动沿用分支名，正文空着，reviewer 一脸茫然。
4. 操作重复：每次都要手动 `git diff` 审阅后才能写 message，效率低下。

目标很明确：让 Agent 读取工作区的实际变更，结合团队规范，产出一份可直接使用的提交信息和分支名，仅保留最后确认权给开发者。

## 构建步骤：从零搭建 Git Agent

### 1. 准备工作：让 OpenClaw 能执行 Git 命令

我们需要一个 MCP 服务（或自定义插件）将 Git CLI 暴露给 Agent。以社区常用的 `git-mcp` 为例，在 OpenClaw 的配置中注册该服务：

```yaml
mcp_servers:
  git-local:
    command: npx
    args:
      - "@anthropic/git-mcp"
      - "--repo-path"
      - "/path/to/your/project"
```

启动后，Agent 就具备了 `git_status`、`git_diff_staged`、`git_diff_unstaged`、`git_log`、`git_create_branch`、`git_commit` 等工具。注意这里需要显式约束权限，避免 Agent 意外执行 `push --force`。

### 2. 定义 Agent 的提交行为

在 system prompt 中注入规范，而不是在每次对话里重复描述。一个工程化的 prompt 片段：

```
你是一个 Git 助手，仅协助处理本地的提交与分支操作。
- 当要求提交时，先使用 git_diff_unstaged 和 git_diff_staged 获取全部变更。
- 根据 diff 内容生成一条 Conventional Commits 格式的提交信息。
- 类型必须从 feat, fix, refactor, perf, test, docs, chore 中选择。
- scope 应为变更涉及的一级模块名，如无法确定则省略。
- 描述要简洁明确，不超过 72 字符，使用英文。
- 生成信息后展示给用户确认，在用户回复“Go”之前不得执行 git_commit。
- 绝对不要改 main 或 master 分支的历史，不允许 force push。
```

这样，每次交互时 Agent 都会遵循同一个契约。

### 3. 实际运行流程

实际对话触发流程示例：

**用户**：帮我提交当前所有修改。

Agent 会依次执行：
1. `git_status` 查看工作区状态。
2. `git_diff_unstaged` 获取未暂存的变更。
3. 若用户未暂存，会主动提醒是否 `git add`。
4. 将 diff 摘要为提交信息，如 `feat(auth): add JWT refresh token rotation logic`。
5. 展示给用户并等待确认。
6. 用户回复“Go”后，Agent 执行 `git_add` (all) 和 `git_commit`。

同理，分支管理可通过 `git_create_branch` 实现：

**用户**：基于 JIRA-3209 创建一个功能分支。

Agent 根据描述生成命名 `feature/JIRA-3209-add-user-export-api`，检查无冲突后创建，并自动切过去。

这个流程的要点是：**AI 负责生成，人类负责审批**，安全边界非常清晰。

## 踩过的坑

### 1. Diff 过大导致 token 爆炸

大型重构的 diff 可能上万行，一次性送给 Agent 会撑爆上下文窗口，且费用不菲。解决方案是在脚本层做预处理：若 diff 行数超过阈值（如 500 行），则只取 `git diff --stat` 和文件列表，让 Agent 基于变动文件和统计摘要生成消息。此时 message 会牺牲部分精确度，但在大改动场景下仍优于“fix stuff”。

### 2. Agent 容易给出过于笼统的信息

即使有 diff，Agent 也可能偷懒生成 `feat: update code`。原因往往是 prompt 缺乏对“具体性”的约束。修正方法是在 prompt 里加入负面示例：

```
错误示例：feat: update logic  （过于笼统）
正确示例：feat(scheduler): add retry backoff for failed webhook deliveries
```

并提供少量 few-shot 示例，稳定性会明显提高。

### 3. Git 操作失败未妥善回调

执行 `git commit` 时若遇到 pre-commit hook 失败（如 lint），Agent 会愣住。需要教会它读取错误输出并反馈给用户，而不是假装成功了。在 MCP 工具的使用说明中强调：任何非零退出码的 stderr 必须原样展示。

### 4. 分支名称冲突

自动生成的分支可能与已有远程分支冲突。需要在创建前执行 `git branch -a` 检查，并在发现冲突时追加序号或随机后缀，同时通知用户手动调整。不能静默覆盖。

## 可复用的工程化建议

- **提交信息模板强约束**：在 Agent 的 prompt 中锁定格式，甚至提供 JSON Schema 式检查逻辑（通过外部工具验证后再提交）。
- **分步确认不可跳过**：永远在 prompt 里写死“生成 → 展示 → 等待人的 Go → 执行”的流转，不能允许 Agent 自主跳过确认。
- **保留原始变更上下文**：每次提交成功后，让 Agent 记录一条简短日志（包含 diff 的文件列表和生成的 message），方便回溯时比对 AI 是否产生了幻觉。
- **把 Agent 集成进编辑器，而非后台自动执行**：目前的可靠性更适合作为 `@agent` 的对话工具，而不是 hook 自动提交。让开发者主动呼唤，是当前阶段最安全的模式。
- **团队共享配置**：将 Agent 的 system prompt、MCP 配置、分支命名规则维护在项目仓库的 `.openclaw/` 目录下，确保所有成员使用的都是同一套契约。

## 总结

用 OpenClaw 构建一个 Git 助手，本质上是在做两件事：**让重复操作可编程**，以及**让规范执行不依赖人的记忆力**。它能处理的不是高深复杂的 Git 场景，而是每天发生几十次的高频动作。这些动作一旦交给 AI，搭配人的最终确认，提交历史的质量会显著提升，分支名不再成为考古痛点。

这不是一个替代开发者的魔术，而是一套需要打磨的工程组合。把规则写清楚，把异常路径补上，把确认环节焊死，你就能得到一个安静、可靠、不会乱 push 的 Git 伙伴。

---

