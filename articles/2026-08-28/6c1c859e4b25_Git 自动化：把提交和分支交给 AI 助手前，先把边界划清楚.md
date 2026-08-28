---
title: Git 自动化：把提交和分支交给 AI 助手前，先把边界划清楚
feedId: 35059
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

最近在 OpenClaw 里把 Git 操作接给 Agent 之后，最明显的变化不是“更快”，而是提交信息终于不再是一堆 `update`、`fix`。之前维护几个小项目时，经常为了写 Conventional Commits 和切分支打断思路。于是尝试让 AI 助手承担四件事：检查工作区、生成分支名、写 commit message、整理 PR 描述。

前提是：给它一套受控工具，而不是直接丢一个 shell。

## 问题

直接给 Agent 一个可执行 shell 很危险。`git reset --hard`、`force push`、`rebase` 这类命令如果被误生成，代价很大。另外，AI 很容易把 `git add -A` 当成默认动作，把临时文件、本地配置、构建产物一起提交。上下文也有限，diff 太大时模型会丢失关键信息，导致提交说明泛化成“update code”。

所以真正要解决的不是“能不能自动提交”，而是“哪些操作可以放权，哪些必须拦截”。

## 做法

我采用“只读工具全开、写操作白名单”的方式。OpenClaw 侧通过 MCP 暴露两组工具。

第一组只读：

- `git status --short`
- `git diff --stat`
- `git diff -- <file>`
- `git log --oneline -20`
- `git branch --show-current`
- `git branch -a`

第二组写操作只保留：

- `git add -- <path...>`
- `git commit -m <message>`
- `git checkout -b <branch>`
- `git push -u origin <branch>`

注意这里没有 `git add -A`，也没有任何 reset、rebase、force push。工具实现层强制要求 `add` 必须传入明确文件路径，不接受空列表或 `./`。

具体配置流程如下：

1. 在 OpenClaw 里增加一个 git MCP server，或者用本地脚本包装成 MCP tool。工具描述里明确：写操作执行前必须返回计划，等待用户确认。
2. 系统提示词加入规则：提交信息使用 Conventional Commits；分支命名 `feature/<issue>-<slug>`；禁止 force push、rebase、commit amend；每次提交前先展示文件清单和 diff 摘要。
3. 工作流：Agent 收到“提交当前改动”后，先跑 `status` 和 `diff --stat`，再根据文件内容生成分支名和 commit message，输出计划，用户确认后调用 `add`、`commit`、`push`。
4. PR 描述复用同一组只读工具：让 Agent 读当前分支相对 main 的 log 和 diff，生成描述草稿，人工编辑后发布。

核心是工具层做参数校验。比如 commit 工具只接受 `type`、`scope`、`description` 三个字段，由工具内部拼成完整 message，避免 AI 注入换行和额外参数。分支工具只允许 `[a-z0-9-/]` 字符，且必须以 `feature/` 或 `fix/` 开头。这样即使模型幻觉，也越不过边界。

## 踩坑点

- **diff 过大导致上下文爆炸**：一次 3000 行 diff 会让模型开始“总结”而不是精确描述。我在工具里对 `git diff` 输出做截断，超过 800 行只保留前 500 行和文件列表，并标记 `truncated`。
- **自动提交污染历史**：之前试过让 Agent 每次任务结束自动 commit，结果出现大量 WIP 提交。后来改成只在用户明确说“提交”时触发。
- **Windows 下路径转义和编码**：Git 输出包含中文文件名时，MCP 工具可能出现乱码。需要在包装脚本里设置 `core.quotepath=false` 和 UTF-8 输出。
- **AI 生成分支名偶发包含空格或大写**：工具层校验拒绝过很多次。规则放在代码里，比放在提示词里可靠。
- **模型对 Git 的理解不稳定**：明确要求它不要执行 `git add -A` 后，偶尔仍会尝试调用不存在的工具。所以提示词里要直接写“你没有 `git add -A` 工具”，而不是“不要用”。

## 可复用建议

1. 写操作一定要工具白名单 + 参数校验，不要只依赖提示词。
2. 提交前让 Agent 展示计划，人工确认一次。对低成本动作来说，多一次确认并不慢。
3. 把规则写进工具描述，而不是只写进系统提示词。工具描述是模型选择参数时的强约束。
4. 保留审计日志：每个写操作记录时间、命令、输入参数、结果。出问题可回滚。
5. 从只读工具开始接入，跑一周只观察，确认模型调用方式稳定后再放开写操作。
6. 配合 pre-commit hook 做最终防线，例如检查提交信息格式、拒绝 `.env` 或构建产物。

## 总结

Git 自动化不是让 AI 替代你的版本控制判断，而是把重复、易错的部分标准化。真正可靠的方案，是把风险关在工具层和规则层，而不是寄希望于模型“自觉”。在 OpenClaw 这类 Agent 环境里，分层授权比炫技更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e1ae378032355fb6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/3c815bb8816cd215.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/d2d65e46fd4ac30f.png)

