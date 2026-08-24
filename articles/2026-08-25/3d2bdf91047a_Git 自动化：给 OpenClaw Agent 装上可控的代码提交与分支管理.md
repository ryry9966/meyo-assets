---
title: Git 自动化：给 OpenClaw Agent 装上可控的代码提交与分支管理
feedId: 34574
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw-CN 社区里，很多朋友已经通过 MCP 或插件把 Agent 接入代码仓库，但多数停留在“读文件、跑命令、解释代码”。真正让 Agent 管理 git 分支、整理提交、推送代码时，问题开始暴露：误提交、切错分支、提交信息混乱、权限过大导致误操作。

我在自己的 OpenClaw 环境里做了一套“人工审批 + 命令白名单”的 Git 自动化，运行一个月后比较稳定。下面记录具体做法和踩坑点。

## 问题

日常开发中有大量机械操作：测试通过后整理 diff、写 commit message、切分支、同步主干、清理旧分支。这些事情规则明确、重复度高，人做容易漏，完全交给 CI 又不够灵活。Agent 适合做这类可审计、可回滚的动作，但前提是权限边界必须清楚。

如果只是给 Agent 一个 shell 工具，让它可以执行任意 git 命令，风险太大。更合理的做法是通过 MCP Server 或在 OpenClaw 插件里封装 git 工具，只暴露有限命令，并让高风险操作必须经过人工确认。

## 做法与步骤

### 1. 配置 Git MCP 工具白名单

我使用的 Git MCP/插件只开放以下命令：

```text
status
diff
diff --staged
add
commit
branch
checkout -b
switch
push
pull --ff-only
log --oneline
```

禁止的命令包括：`push --force`、`reset --hard`、`clean -fd`、`branch -D`、删除远程分支等破坏性操作。

在 OpenClaw 的工具策略中，把这些 git 工具单独配置为“写操作需要确认”。尤其是 `push`、`branch` 删除类操作，必须走审批。

### 2. 固定 Agent 行为约束

在 Agent 的 preamble 或系统提示里写死一套规则：

- 只负责指定仓库，默认在 `feature/` 或 `fix/` 前缀分支工作。
- 任何修改前先运行 `git status` 和 `git diff --stat`，并打印当前分支。
- 提交信息使用 conventional commits：`type(scope): subject`，subject 不超过 60 字符。
- 一次提交只包含同一逻辑变更；不确定时先列出计划，等待确认。
- 不允许自行 push 到 `main` 或 `master`。

这套约束比让模型自由发挥稳定很多。

### 3. 实际工作流

以处理 issue 为例：

```text
OpenClaw：读取 issue #42，创建分支 fix/issue-42，修改文件，运行测试，分段提交并 push 到 origin。
```

Agent 的执行顺序通常是：

1. `git status` + `git log --oneline -5` 确认当前上下文
2. `git checkout -b fix/issue-42`
3. 修改代码后运行测试
4. `git diff --stat` 判断改动范围
5. 按文件组织提交，生成多个 commit，例如：
   - `fix(parser): handle empty input`
   - `test(parser): add regression cases`
   - `chore(ci): update test timeout`
6. 推送前在消息中附上即将执行的命令和 diff 摘要，等待人工批准

### 4. 设置人工确认 Gate

在 OpenClaw 中，我给 `push`、`branch delete` 配置了 `require_approval`。Agent 完成本地提交后，会停下来展示：

```text
即将执行：git push origin fix/issue-42
变更摘要：3 files changed, +87 -12
```

用户回复“批准”后才会真正推送。这样即使 Agent 提交结果不理想，也来得及修改。

## 踩坑点

**工作区污染**
Agent 很容易把 `.env`、`node_modules`、临时脚本一起 `git add`。必须在仓库根目录维护好 `.gitignore`，并在 prompt 中限定添加白名单，例如只允许 add `src/`、`tests/`、`docs/` 下的文件。提交前先人工确认 `git diff --name-only`。

**提交粒度过大**
一次 commit 混杂格式化、功能修改和依赖升级。解决方法是要求格式化作为独立 commit，功能修改按目录或文件分组。如果文件多，先让 Agent 输出提交计划，不要直接连续提交。

**上下文爆炸**
大型 diff 直接塞给 LLM 会迅速消耗 token。先看 `git diff --stat`，再按文件逐个 diff。如果单个文件 diff 超过 300 行，让 Agent 先输出摘要，请求确认后再继续。

**并发冲突**
两个 Agent 同时操作同一仓库，分支和工作区会互相干扰。实践下来，比较稳妥的是给每个 Agent 使用独立的 `git worktree`，或者加一个简单的文件锁（如 `.openclaw-lock`），拿到锁才能执行写操作。

**权限失控**
如果 MCP 暴露了 shell 级 git，Agent 可能误执行 `git reset --hard` 导致工作丢失。所以一定要用命令白名单，并在工具描述里明确“不会执行破坏性命令”。这是最后一道防线，不能省。

## 可复用建议

- **dry-run 模式**：前两周先让 Agent 只生成命令列表和理由，不实际执行。人工确认无误后再放开写权限。
- **提交信息模板**：把 conventional commit 类型和 few-shot 示例放进 prompt，比让模型自由发挥稳定得多。
- **审计日志**：把 Agent 执行的每条 git 命令写到 `logs/agent-git.log`，便于回滚和定位问题。
- **分支生命周期**：定期让 Agent 列出已合并分支，生成建议删除列表，但删除操作必须人工点击。
- **测试前置**：要求 Agent 在 commit 前跑本地 lint/test，失败则不允许提交。可以配合 pre-commit hook 做双保险。

## 总结

Git 自动化不是让 AI 全权接管仓库，而是把整理 diff、写 commit、建分支这类可审计的机械动作交给 Agent，把 push、合并、删除等高风险动作留给人。只要权限白名单和人工确认 gate 做扎实，AI 管理 Git 的可靠性比想象中高，出错时也容易定位。建议从只读和 dry-run 开始，逐步放开写权限，不要一上来就全自动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/77cdc1dc1d8dd538.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/479e8fb61d4825b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/b4331417b45787b7.png)

