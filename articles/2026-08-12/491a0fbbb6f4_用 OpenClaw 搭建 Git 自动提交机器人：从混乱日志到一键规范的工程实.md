---
title: 用 OpenClaw 搭建 Git 自动提交机器人：从混乱日志到一键规范的工程实践
feedId: 32698
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

小团队或个人开发者经常面临这样的现实：功能写完了，`git add . && git commit -m "update"` 一气呵成。等到要回溯变更、生成 changelog 或做代码 review 时，满屏的 `fix`、`update`、`temp` 让人无从下手。虽然市面上有 `commitlint`、`husky` 等工具能约束提交格式，但它们无法帮开发者**总结变更内容**，更不会主动管理分支。

如果你已经在用 OpenClaw 这类 Agent 框架做任务编排，可以试着把 Git 的日常操作也交给它。本文记录一次工程化实践：利用 OpenClaw 的 MCP 工具调用能力和简单的状态机，实现 diff 驱动的自动提交信息生成与分支创建，让 AI 充当“会写规范日志的 git 助手”。

## 问题拆解

将 Git 操作自动化拆成两个核心任务：

1. **理解变更**：从 `git diff` 中提取有意义的改动摘要，生成符合团队规范的 commit message。  
2. **执行操作**：在合适的时机执行 `git add`、`git commit` 和分支切换，且不能对仓库造成不可逆伤害。

其中第一个任务是对 LLM 的自然语言理解考验，第二个则要解决安全沙箱与错误处理。

## 实现步骤

### 1. 环境与工具准备

- OpenClaw 运行环境（已配置好 Agent 与 MCP 客户端）
- 一个 Git 仓库，安装 `git` 命令行
- 为 Agent 准备一个专用 Git 用户，避免污染全局配置
- 可选：`git-filter-repo` 或 pre-commit 钩子作为兜底防护

关键组件是 **MCP Git Server** — 在 OpenClaw 中通过 MCP 连接本地命令行 Git，封装了 `git_diff_staged`、`git_commit`、`git_create_branch` 等工具。

### 2. 设计 Prompt 与变更摘要

不直接把整个 diff 扔给模型，而是分步处理：

**步骤 A：筛选变更范围**  
用 `git diff --stat` 先获取改动文件列表及行数统计，交由 Agent 判断是否需要生成提交。如果改动全部在某自动生成目录（如 `dist/`），则直接跳过。

**步骤 B：分段获取 diff 内容**  
对于大型 diff，按文件切分，每个文件 diff 限制不超过 150 行，防止超出 token 窗口。使用规则：跳过 lock 文件、二进制文件，仅处理文本变更。

**步骤 C：生成 commit message**  
用结构化 Prompt 规定输出格式：

```
根据以下 git diff 内容生成一段符合 Conventional Commits 规范的提交信息。
格式：<type>(<scope>): <简短描述>
类型从 feat, fix, docs, refactor, test, chore 中选择。
描述用英文，不超过 72 字符。
若 diff 涉及多个不相关改动，建议拆分为多个独立提交。
```

让模型输出纯文本，不做 Markdown 包裹，方便直接作为参数传入。

### 3. 自动化工作流

在 OpenClaw 中定义一个 **GitAutoAgent**，按以下状态流转：

- **IDLE** → 用户触发（可定时或通过 webhook）
- **ANALYZE** → 运行 `git diff --cached`（只处理暂存区，保护未暂存的脏工作区）
- **GENERATE** → 逐个文件分析，生成 commit message 候选，交由用户确认
- **EXECUTE** → 调用 `git_commit` 工具，附带 `-m` 参数
- **BRANCH** → 若 Agent 判断需要新建分支（如检测到 feat 类型），则基于当前分支创建 `feat/<desc>` 并切换

执行过程完全透明，所有操作记录到 OpenClaw 的 audit log 中。

### 4. 控制权限与风险

- 仅允许 Agent 在 **非保护分支** 上操作，用 Git hook 在服务端做第二次拦截。
- 禁止 force push、禁止删除远程分支的权限。
- 所有 commit 前执行一次 `git diff --cached` 二次校验，避免幻觉导致误提交。
- 首次运行先用 `dry-run` 模式，只输出准备执行的命令，不真正改动仓库。

## 踩坑记录

### Git 指令输出被截断
`git diff` 输出包含 ANSI 颜色转义码，通过 `--no-color` 去除。同时设置 `core.pager=cat` 确保无分页干扰。

### Token 消费爆炸
一个 `package-lock.json` 的 diff 就能吃掉几万 token。必须用 `.gitignore` 风格的规则预先过滤，定义 `allowed_extensions` 白名单或 `exclude_patterns` 黑名单。

### 自动生成的提交信息过于泛化
问题出在 prompt 缺乏上下文。解决方案是让 Agent 读取最近 3 条历史提交，作为生成时的风格参考。在 prompt 中加入：

```
参考该仓库近三次提交风格：
1. feat(auth): add JWT token refresh
2. fix(api): correct pagination offset
...
```

### 分支命名冲突
自动生成分支名可能重复，如 `feat/update-readme`。需要加入随机后缀或序号检测：如果分支已存在，询问用户是否变更为 `feat/update-readme-2` 或直接拒绝操作。

## 可复用建议

1. **渐进式接入**：从仅生成 commit message 但不提交开始，让开发者用两周时间验证生成质量，再开放自动提交权限。
2. **人工审批节点**：对于 feat、refactor 类的提交，要求用户在终端确认，避免语义理解偏差。
3. **维护自定义规则库**：将需要跳过的文件模式、scope 缩写映射表写成 YAML 配置，供 Agent 每轮对话加载，而不是硬编码在 prompt 中。
4. **定期回顾**：每月随机抽取 20 条自动生成的提交信息做人工评审，根据反馈微调 prompt 和预处理规则。
5. **兜底方案**：始终保留 `--dry-run` 模式，结合 `git reflog` 可随时回退误操作。

## 总结

让 AI 助手接管 Git 提交的初衷不是取代开发者，而是把“写规范提交信息”这种低脑力但高频的动作系统化。通过限制上下文范围、设置人工卡点和全套日志，能在不牺牲安全性的前提下显著提升日常提交质量。工程化之后，你甚至会发现它还能被动地帮你发现偶发的拼写错误或忘记更新的文档引用——因为 diff 从不撒谎。

自动化并没有魔法，只是把规则、校验和人机协作拧成了一股绳。

---

