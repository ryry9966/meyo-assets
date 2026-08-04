---
title: Git 自动化：用 OpenClaw 把提交和分支交给 AI 助手
feedId: 31658
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

多数人的 Git 工作流是这样的：`git add`、`git commit`、`git push`，然后祈祷提交信息写得足够清楚。但实际情况下，commit message 往往要么是 `update`，要么是 `fix bug`，三个月后回看日志，没人记得这段代码当初为什么改。

我在 OpenClaw 社区里看到不少朋友已经在用 MCP 工具链调模型、跑 Agent，但 Git 操作仍然停留在手动敲命令的阶段。其实 Git 是一个非常适合交给 AI 的自动化场景：规则明确、行为可预期、出错可回滚。

## 问题

手动管理 Git 的核心痛点有三个：

1. **提交信息质量低**——纯手写时人倾向于偷懒，一行 `fix` 就完事
2. **分支操作频率高但决策成本低**——创建、切换、合并、删除，这些操作模式固定，但占了日常不少时间
3. **AI 工具与 Git 集成断层**——即使有 Agent 环境，也没有让 AI 实际参与到版本管理流程里

换句话说，AI 完全有能力做这件事，但默认工作流里压根没给它机会。

## 做法：三个自动化层次

### 层次一：用 MCP 给 AI 装上 Git 手

如果你用的是 OpenClaw，最直接的方式是挂一个 Git MCP Server。这样 Agent 就拥有对仓库的读写能力，包括查看状态、diff、日志、创建分支、提交、推送。

```json
{
  "mcpServers": {
    "git": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-git"]
    }
  }
}
```

配置完成后，你不需要自己敲 `git log --oneline -10`，直接告诉 AI：“看看最近提交情况，告诉我项目的稳定节奏”，它会自己调工具去查。

### 层次二：让 Agent 生成结构化提交信息

这是最实用的一层。过去我用自研脚本跑 `gitmoji` 加 conventional commits 模板，但效果有限，因为脚本只用正则匹配 diff 里的关键字。换成 LLM 之后，提交信息质量提升了一个维度。

我常用的做法是维护一个 commit 规范文件（类似 `.git-commit-template`），告诉 Agent 的 system prompt：

> 每次生成 commit message 时，先 `git diff --stat` 看改动范围，再 `git diff` 看具体内容。消息格式遵循 Conventional Commits，type 从 feat/fix/refactor/docs/test/chore 中选择，并说明变更动机（why），不重复代码本身做了什么（what）。

实际效果是，Agent 生成的提交信息能准确表达“为什么删掉了这个函数”、“为什么调整了这个配置”，而不是机械罗列文件名。

### 层次三：分支清理与自动化巡检

在一个迭代周期内，分支数量会快速增长。我让 OpenClaw 定时跑一个巡检任务，逻辑很简单：

- 列出所有本地分支
- 对比 `main` 分支，找出已合并的分支
- 询问是否删除（而不是直接删）
- 对未合并但超过 30 天不活跃的分支打标记，提醒我处理

这里的关键点是：**AI 可以做分析和建议，但删除操作必须经人确认**。这不是能力问题，而是责任边界。

## 踩坑记录

这套方案跑了大约两个月，最值得记录的坑有三个：

**坑一：AI 看到的 diff 与真实上下文脱节**

Agent 看 diff 时只看到改动，看不到分支语义。比如某个分支叫 `hotfix/login-timeout`，但 diff 里改的是一段工具函数。AI 生成的 message 可能偏离实际意图。解决办法是让 Agent 先查看当前分支名和关联的 issue，再生成信息。

**坑二：自动 commit 会打断开发流**

有过一次事故：我在编码中途，Agent 检测到文件变化就直接提交了。半成品代码被推到远程，CI 直接红了。后来的修复是加了一个“门卫”——只有检测到稳定的空闲状态（比如 5 分钟无文件变更）才允许自动提交，且只做本地 commit 不 push。

**坑三：hook 与 MCP 的协作冲突**

我原本配了 `pre-commit` hook 跑 lint，但 Agent 通过 MCP 提交时，某些 hook 的路径环境没有加载，导致 hook 直接失败。看起来是 Agent 不会 `nvm use`，实际是 MCP server 启动时没有继承 shell 环境。解决方式是让 MCP server 用固定 Node 版本启动，或者把环境变量写进 server 的启动命令里。

## 可复用建议

1. **不要一开始就做全自动**。前两周只让 Agent 做 commit message 生成和分支分析，人工执行命令
2. **写入规范比写入提示词更可靠**。把 commit 规范维护在仓库根目录，Agent 先读文件再干活
3. **所有破坏性操作加确认门槛**。删除分支、强推、回滚这类操作，强制要求 Agent 先输出计划，再由人确认
4. **给 Agent 一个 Git command 白名单**。比如 `add/commit/log/diff/branch` 放行，`push/reset/rebase` 需额外授权
5. **保留手动逃生通道**。即使 Agent 挂着，也要保证 `Ctrl+C` 和终端敲命令永远可用

## 总结

Git 自动化不是让 AI 替代你管理代码，而是把“格式化的重复劳动”交给 AI，把“决策”留给人。目前这套方案大概能省掉每天 20~30 分钟的提交整理时间，更重要的是，提交历史从此变得干净、可追溯。

没有银弹，但值得一试。下一次当你准备敲 `git commit -m "update"` 的时候，值得想一想：这个动作，能不能让 AI 来做？

---

