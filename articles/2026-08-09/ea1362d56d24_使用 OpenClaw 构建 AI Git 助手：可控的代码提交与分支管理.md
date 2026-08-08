---
title: 使用 OpenClaw 构建 AI Git 助手：可控的代码提交与分支管理
feedId: 32166
source: 综合讨论
publishedAt: 2026-08-09
---

# 使用 OpenClaw 构建 AI Git 助手：可控的代码提交与分支管理

## 背景：为什么需要 AI 接管重复的 Git 操作

日常开发中，高频且机械的 Git 操作（暂存、提交、切分支）占据了大量心智带宽。这些操作本身并不复杂，但一旦数量多起来，就容易出现 `commit message` 随意、分支命名不规范、忘记推送等问题。尤其在多任务并行、频繁上下文切换时，手动操作 Git 不仅低效，还容易出错。

另一方面，本地 AI 助手（如基于 OpenClaw + MCP 协议的工具链）已经能安全地操作文件、运行脚本。如果能将一部分 Git 工作交给 AI 来处理，就可以把开发者从重复劳动中解放出来，同时让代码仓库保持更高的规范性。

不过，直接把 Git 控制权交给 AI 是很危险的。没有约束的 AI 可能会执行 `git push --force`、错误回滚甚至删除分支。因此，我们要解决的问题不是“能不能做”，而是：“**如何让 AI 像一位靠谱的协作者一样，只在安全边界内操作 Git，并始终需要你的确认**”。

本文将基于 OpenClaw 和 MCP Git 服务器，搭建一个 **AI Git 助手**，让它能帮你生成高质量的 commit message、按规则创建分支，并避免任何危险操作。

## 做法：四步搭建可控的 Git 自动化

### 1. 准备工作：MCP Git 服务器

我们需要一个符合 MCP 协议的 Git 工具服务器，这样 OpenClaw 就能通过标准接口调用 Git 命令。社区有一个常用的实现 `mcp-git-server`，它提供了 `git_status`、`git_diff_unstaged`、`git_add`、`git_commit`、`git_branch`、`git_checkout` 等工具。

```bash
# 安装 MCP Git 服务器（Node.js 环境）
npm install -g @anthropic/mcp-server-git
```

启动时会自动扫描当前工作目录的 Git 仓库，并暴露相关工具。**关键点**：该服务器默认会暴露所有 Git 子命令，但我们在 OpenClaw 配置里可以过滤，只允许安全的工具白名单。

### 2. 配置 OpenClaw：仅授权安全工具

在 OpenClaw 的 MCP 配置中，为 Git 服务器建立一个最小权限的 session。我们只允许以下工具：

- `git_status`
- `git_diff_unstaged`
- `git_diff_staged`
- `git_add`
- `git_commit`
- `git_branch` (仅 `--list` 和 `--create` 分支）
- `git_checkout` (仅 `-b` 新建分支)

**明确禁止** `git_reset`、`git_rebase`、`git_push`、`git_merge` 等可能影响远端或历史的操作。在 OpenClaw 中可以这样配置 `mcp_settings.json`：

```json
{
  "mcpServers": {
    "git-assistant": {
      "command": "mcp-server-git",
      "args": ["--repository", "/path/to/your/repo"],
      "allowedTools": [
        "git_status",
        "git_diff_unstaged",
        "git_diff_staged",
        "git_add",
        "git_commit",
        "git_branch_create",
        "git_checkout_new_branch"
      ]
    }
  }
}
```

如果你的 MCP 实现不支持 `allowedTools`，则可以写一个包装脚本或通过系统 prompt 进行行为约束（见下一步）。

### 3. 定义 AI 助手的系统提示词

即使工具列表已受限，AI 仍可能用合法工具做出不当行为（例如把不相关的文件一起 add、或写出误导性的 commit message）。因此，系统提示词是最后一道防线，需要描述清楚 AI 的角色和行为边界。

我们设定一个专门的 “Git Assistant” agent，它的固定 prompt 可以包含如下内容：

> 你是一个谨慎的本地 Git 助手。你只被允许执行以下操作，并且必须遵守约束：
> - 使用 `git_status` 查看仓库状态，用 `git_diff_unstaged` 查看未暂存的变更。
> - 在收到明确的“提交”指令后，分析 diff 内容，生成符合 Conventional Commits 规范的 commit message，然后执行 `git_add` 和 `git_commit`。
> - 绝不要添加二进制文件、node_modules、.env 等通常被 `.gitignore` 排除的文件。
> - 当用户要求创建分支时，根据当前任务或 issue 自动生成合适的分支名（如 `feat/xxx`、`fix/xxx`），然后执行新建并切换。
> - 你无权执行任何可能改变历史或推送至远端的操作。如果需要，必须询问并让用户手动完成。

这个 prompt 还会帮助 AI 在生成 commit message 时遵循规范，例如要求以 `feat:`、`fix:`、`chore:` 开头，并附上简要说明。

### 4. 实际使用流程

配置完成后，开发者可以在 OpenClaw 的对话界面中，直接用自然语言触发 Git 操作：

- “帮我看看当前仓库状态”
- “这几个文件我改完了，生成一个合适的 commit message 并提交”
- “根据 issue #42 创建一个新分支来处理这个功能”

AI 会拉取 diff，分析变更意图，生成 message **并展示给你看**，你输入 `yes` 或 `confirm` 后它才真正执行 `git_add` 和 `git_commit`。这种半自动的确认机制，是保证安全性的关键。你也可以让它直接执行，但建议初期的每次提交都保留人工确认。

## 踩坑与排障

实际落地过程中，有几个容易踩的坑：

**坑 1：MCP 服务器的工作目录不正确**
如果你在 OpenClaw 中启动 Git 服务器时，当前工作目录不是 Git 仓库，或者跨容器路径映射有问题，会直接报 `not a git repository`。务必检查 `args` 中的 `--repository` 参数，使用绝对路径，并确保运行 OpenClaw 的用户有读写权限。

**坑 2：commit message 生成质量飘忽不定**
对于较大 diff，AI 容易生成泛泛的 “update code” 这种无用信息。解决办法：在系统 prompt 中补充 few-shot 示例，或要求 AI 先列出变更意图再生成 message。如果使用本地模型，可以尝试在 diff 较大的时候先让 AI 输出摘要，经你确认后再生成正式 message。

**坑 3：意外暂存不想提交的文件**
即使告诉 AI 不要 add 二进制文件，它仍可能因为 `git add .` 而犯错。务必将 `git_add` 的参数限制为逐个文件或仅 `-u`，不要直接允许 `-A` 或 `.`。更好的做法是让 AI 先读出具体变更文件列表，由你口头选择要 add 哪些，它再逐一执行。

**坑 4：分支命名冲突**
如果仓库已存在 `feat/xxx` 分支，AI 直接 `git checkout -b` 会失败。可以预先在 prompt 中要求它先用 `git_branch --list` 检查重名，再生成备选名称。

## 可复用建议

- **环境隔离**：建议先在非生产分支上实验，或者克隆一个临时仓库来体验 AI 的 Git 操作，避免误操作影响主要工作。
- **分层权限**：日常提交由 AI 完成，但推送、合并等操作始终保留在人类手中。可以用 Git hooks（如 `pre-push`）作为额外安全层。
- **Commit 规范模板**：在项目根目录添加 `commit-convention.md`，让 AI 引用该文件作为生成 message 的依据，能大幅提升一致性。
- **成本控制**：如果调用云端大模型，diff 内容可能很大，烧 token 快。建议先让 AI 读取文件差异摘要（按文件而非全量 diff），或者限制单次请求的最大 diff 行数。

## 总结

借助 OpenClaw 和 MCP Git 服务器，我们把 Git 中最枯燥的部分交给了 AI，同时通过工具白名单、系统提示词和人工确认三道防线，把风险控制在可接受范围内。最终效果是：commit message 总是规范的，分支命名不再随意，开发者只需要对变更内容做出决策，机械操作完全由 AI 执行。

这套模式还可以扩展到更多自动化场景，比如自动生成 CHANGELOG、根据标签创建 release 分支等。核心思路不变：**让 AI 做你手里可控的工具，而不是一个无脑的 root 机器人**。

---

