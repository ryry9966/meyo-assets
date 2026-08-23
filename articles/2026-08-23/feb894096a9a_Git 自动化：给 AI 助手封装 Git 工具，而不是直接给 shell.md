---
title: Git 自动化：给 AI 助手封装 Git 工具，而不是直接给 shell
feedId: 34345
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw/Agent 类项目里，很多重复劳动发生在 Git 操作上：看状态、整理 diff、写 commit message、切分支、推送前检查。让 AI 助手参与这些操作，理论上能省时间，但 Git 是一个“不可逆、上下文相关、容易误伤”的领域。直接给 Agent 一个 shell 执行 `git add . && git commit -m ...`，翻车概率很高。

## 问题

实践中常见几类故障：

- AI 把 `.env`、`secret.json`、本地配置误加入提交。
- 在 main/master 分支直接 commit，污染主干。
- 生成的 commit message 不规范，或者与实际 diff 不匹配。
- Agent 循环提交：提交后状态未变化，仍然继续生成新提交。
- 操作不可审计，出问题难回滚。

所以需要把 Git 能力包装成带边界和校验的工具，而不是开放任意命令执行。

## 做法

### 1. 封装只做一件事的 Git 工具

建议至少提供这些工具：

- `git_status`：输出 `git status --porcelain=v1`。
- `git_diff`：输出 `git diff --stat` 和按文件截断的 diff。
- `git_diff_staged`：用于提交前复审。
- `git_add(paths: string[])`：只允许显式路径，不提供 `git add .`。
- `git_commit(message: string)`：仅提交已暂存内容。
- `git_create_branch(name: string, base?: string)`：创建并切换分支。
- `git_switch_branch(name: string)`：切换分支，但拒绝在脏工作区切换。
- `git_log(limit: number)`：查看历史。
- `git_push_dry_run`：展示将要推送的 commit 和远程 diff。

每个工具内部做参数校验。例如 `git_add` 路径必须在仓库根目录内，且不能用 `..`；路径如果命中敏感文件模式，直接拒绝。

### 2. 让 AI 先出计划，再执行

不要直接把用户指令转成 Git 命令。流程分两步：

1. Agent 调用 `git_status` 和 `git_diff`，理解当前工作区。
2. Agent 输出一个操作计划，比如：

```text
Plan:
- add: src/agent/git_tools.py, tests/test_git_tools.py
- commit message: feat(git): add bounded git tool wrappers
- branch: leave on current branch feature/git-auto
```

用户确认或工具配置允许后，再调用写操作。这个“计划-确认-执行”的机制，能避免大部分误提交。

### 3. 约束 commit message

给 Agent 的 prompt 里明确要求 Conventional Commits，并说明只基于 `git_diff_staged` 生成，不能编造。最好在 `git_commit` 工具内接入校验，例如用 `commitlint` 规则；校验失败不执行，并返回错误信息让 Agent 重写。

示例工具返回：

```json
{
  "ok": false,
  "error": "commit message rejected by commitlint: subject must not be empty"
}
```

这比事后跑 CI 检查更早暴露问题。

### 4. 分支策略

限制 AI 只能操作 `feature/`、`fix/`、`chore/` 等前缀分支。禁止在默认分支直接提交。`git_create_branch` 可以内置规则：

- `base` 不指定时，默认从最新 main 创建。
- 创建前检查工作区是否干净，不干净则拒绝。
- 新分支必须匹配 `^(feature|fix|chore)/[a-z0-9-]+`。

### 5. 操作日志与回滚

给每次写操作记录 JSON 日志，包括时间、工具名、参数、结果、当前 commit hash。出问题时可以从日志定位到具体操作，而不是只看到“AI 又乱搞了”。

## 踩坑点

1. **敏感文件误提交**：即使 `.gitignore` 存在，`git add paths` 如果显式传入敏感路径仍会加入。工具内需要维护一份敏感模式列表，如 `*.env`、`.env.*`、`*secret*`、`*.pem`、`config/local*`。
2. **长 diff 打爆上下文**：不要让 `git_diff` 一次性输出全量 diff。按文件截断，比如每文件最多 200 行，总输出不超过 8000 字符，并先给 `--stat` 总览。
3. **循环提交**：Agent 提交后如果工作区仍有变化，可能会继续提交。设置单次任务最大写操作数，比如最多 3 次 commit；为空 diff 时直接返回 `nothing to commit`。
4. **脏工作区切换分支失败**：`git switch` 会因为未提交变更报错。工具应预检查 `git status --porcelain`，如果有未暂存变更，要求 Agent 先提交或暂存。
5. **push 不可逆**：远程操作一定要分开。只允许 `git_push_dry_run`，真正的 push 由用户在终端执行，或至少需要额外确认。千万不要给 AI 开放 `push --force`。

## 可复用建议

- **不要暴露 raw shell**：用工具接口代替，这样权限、参数、日志都可控。
- **读写分离**：只读工具可以自动执行，写操作默认需要确认。
- **接入现有 hook**：commitlint、pre-commit、pre-push 等 hook 不要因为自动化而绕过，它们反而是最后一道防线。
- **把 AI 当作草稿生成器**：commit message 和分支名可以先生成，人在 PR 流程中做最终确认。
- **从低风险场景开始**：先自动化 `git status` 摘要、commit message 建议，再逐步放开分支创建和暂存操作。

## 总结

Git 自动化不是把 shell 交给 AI，而是把 Git 能力拆成受控工具，加上计划-确认-执行、敏感文件过滤、commit message 校验和操作日志。这样 AI 助手才能真正帮助管理提交和分支，而不是制造新的排查负担。对于 OpenClaw 用户来说，这套封装可以直接作为 MCP server 或自定义工具落地，投入成本不高，但能明显减少日常 Git 重复劳动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/11b67c923875d52c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ae24ae2a16d8213b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/365e6a4133f70309.png)

