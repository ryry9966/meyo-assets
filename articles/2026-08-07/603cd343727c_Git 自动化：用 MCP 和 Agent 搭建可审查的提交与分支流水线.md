---
title: Git 自动化：用 MCP 和 Agent 搭建可审查的提交与分支流水线
feedId: 31931
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：每天都在重复的 Git 仪式

写代码的节奏常常被 Git 操作打断：改完一个函数，需要想一句像样的 commit message；修完 bug 准备切分支合并，命名纠结半天；提 PR 前还要 rebase 最新的 main，再解决一遍冲突。这些操作虽然简单，但频率极高，累加起来消耗了大量注意力。

更麻烦的是，在多人协作或快速迭代的项目里，提交信息不规范、分支命名随意、遗漏推送、合并步骤错误，会直接拖慢 Code Review 效率，甚至引发线上事故。于是自然想到：能不能把这些规律性强、判断标准相对清晰的 Git 任务，交给 AI 助手去执行，同时保留人工审查的入口？

本文不讨论“全自动 AI 程序员”，而是聚焦一个务实的工程实践——**在 OpenClaw 环境中，通过 MCP 暴露 Git 能力，组合 Agent 插件，构建一条可审查、可干预的半自动 Git 流水线**。它负责生成规范提交信息、创建分支、执行常规合并，并在关键节点停下来等你确认。

## 问题拆解：要让 AI 操控 Git，需要解决什么

要让一个 LLM 替你做 Git 操作，至少需要三样东西：

1. **安全的工具通道**：不能直接把 shell 暴露给模型，必须有粒度受控的 Git 能力接口。
2. **可理解的上下文**：Agent 需要能读取 `git diff`、变更文件列表、当前分支状况，甚至关联的 issue 描述。
3. **可校验的输出**：AI 生成的 commit message 或分支名必须经过规则校验，不符合约定的直接拒绝或要求重试，而不是直接执行。

OpenClaw 生态恰好提供了组合方案：MCP（Model Context Protocol）负责将 Git 操作封装为标准化工具；Agent 插件通过 System Prompt 和外部知识约束行为；内置的动作审批机制可以在高危操作前暂停并等待人工介入。

## 实现步骤：从 diff 到 push 的半自动管道

以下是一个已经跑在内部项目上的实现路径，你可以按需裁减。

### 1. 搭建 Git MCP 服务器

选用社区维护的 `git-mcp` 服务器（例如通过 `npx` 直接启动），在 MCP 配置中仅暴露必须的工具，并限定工作目录：

```json
{
  "mcpServers": {
    "git": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-git", "--repository", "/path/to/repo"],
      "allowedTools": [
        "git_status",
        "git_diff_unstaged",
        "git_diff_staged",
        "git_add",
        "git_commit",
        "git_branch",
        "git_checkout",
        "git_push"
      ]
    }
  }
}
```

注意：**永远不要暴露 `git push --force`**，不要暴露 `git reset --hard` 等不可逆命令。安全是自动化 Git 的生命线。

### 2. 设计 Agent 的 Prompt 与规则

在 OpenClaw 中创建一个专用技能，命名为 `git-auto`。System Prompt 关键片段：

```markdown
你是 Git 操作助手。你需要根据用户意图，使用提供的 MCP 工具完成 Git 操作。
- 所有 commit message 必须符合 Conventional Commits 格式：`type(scope): subject`，type 只能是 feat, fix, docs, refactor, test, chore。
- 分支命名规范：`feature/<issue-id>-<slug>`, `bugfix/<issue-id>-<slug>`, `hotfix/<issue-id>-<slug>`。
- 在提交前，必须向我展示完整的 diff 摘要和拟提交信息，等待我回复“OK”后才能执行 `git_commit` 和 `git_push`。
- 遇到冲突时，只生成冲突报告，不要尝试自动解决。
```

然后，将 `git-auto` 技能绑定一个动作触发器，比如收到 `/commit` 命令，或监听指定 Git 事件（如果你在本地跑了一个 daemon）。

### 3. 实际流程演示

假设你刚修改了登录模块，运行命令：

```
/commit 修复 LinkedIn OAuth 回调处理
```

Agent 会先执行 `git_status`、`git_diff_unstaged`，拿到变更细节。接着用 LLM 生成符合规范的提交信息，例如：

```
fix(oauth): handle missing state parameter in LinkedIn callback
```

它会将 diff 摘要和这条 message 打印出来，并阻塞等待你的确认。你回复 `OK` 后，Agent 依次执行 `git_add`、`git_commit`、`git_push`。

分支创建的流程类似。若你说：

```
/branch feature 1234 add payment retry logic
```

Agent 会检查当前分支，切换到 main 并 pull 最新，然后执行：

```
git checkout -b feature/1234-add-payment-retry-logic
```

这样分支名和 issue 严格关联，后续自动生成 PR 标题也方便。

### 4. 高级模式：合并与 rebase 的半自动处理

定期 rebase 是很多人的痛点。我们可以为 `git-auto` 增加一个子技能 `rebase-main`，动作如下：

1. 检查工作区是否干净，不干净则报错退出。
2. 切换到 main，执行 `git pull`。
3. 切回原分支，执行 `git rebase main`。
4. 若成功，输出提示；若出现冲突，逐文件列出冲突状态，暂停并要求手动解决。

这里的关键是 **永远不自动 force push**。rebase 后的首次推送需要你明确指定 `--force-with-lease`，Agent 只能建议，不能擅自动手。

## 踩坑点：别让 AI 成为新的混乱源

在实际落地中，下面几个坑非常普遍：

- **并发操作导致 .git 锁**：如果你同时有多个 Agent 或手动操作，很容易出现 index.lock 冲突。建议使用文件锁或集中排队机制，最简单的做法是只在无其他 git 进程时才允许 Agent 执行写操作。
- **生成内容失控**：LLM 偶尔会产生拼写错误、不存在的 scope，甚至插入解释性文字。务必在后处理阶段用正则校验，不通过的直接拒绝并让 Agent 重试。比如强行截断多行 description。
- **通过 shell 注入？**：使用 MCP 工具传递参数可以避免直接拼接 shell 命令。只要你不把 `git_commit` 的 message 参数拼进 `-m` 字符串（工具内部会使用 `--file` 或参数数组），风险就很小。
- **敏感信息泄漏**：Agent 可能读到包含密钥的 diff（即使是测试代码）并生成明文提交信息，进而推送到公开仓库。必须配置 pre-commit hook 或 diff 过滤模板，对匹配 SECRET 正则的变更块做隐藏或禁止提交。
- **人工审查疲劳**：如果每次都要确认，自动化价值会打折扣。可以建立信任分级：对于只触及文档、测试文件的变更，允许跳过人工确认；对业务逻辑或配置文件修改，必须强制确认。

## 可复用建议：一份生产级配置的轮廓

根据多次折腾的经验，建议将自动化 Git 的能力打成一个可配置的 OpenClaw 插件包，内部包含：

- `mcp-servers.json`：定义 Git MCP 服务器及其允许工具列表。
- `skills/git-auto.md`：主技能 Prompt，符合上述规则。
- `scripts/pre-commit-police.sh`：校验生成的 commit message 是否符合 Conventional Commits 格式，可由 Agent 调用或作为 hook。
- `config.yaml`：定义哪些目录下的变更可免人工审核，哪些必须双签。

同时，建立一份团队公约：

> 自动化 Git 生成的所有分支、commit，必须关联 issue 编号。PR 标题建议由 Agent 生成，但描述必须手写，以确保业务意图清晰。

这个约定能极大降低 AI 误操作带来的沟通成本。

## 总结

用 AI 助手管理代码提交和分支，并不是为了把 Git 变成黑盒，而是用程序化方式收敛操作的一致性，从而将有限的注意力留给真正需要人类判断的事情：架构决策、安全边界和业务逻辑。

MCP 提供了受控的工具暴露方式，OpenClaw 的 Agent 框架可以让整个流程既智能又安全。从生成规范 commit message 开始，到自动分支创建与 rebase，每一步都可以逐步引入，逐步建立起信任。记住一个底线：**AI 负责提议和准备，人类负责确认和最终执行。** 这样你的 Git 历史会整洁许多，而你不会在凌晨三点排查 rebase 事故。

---

