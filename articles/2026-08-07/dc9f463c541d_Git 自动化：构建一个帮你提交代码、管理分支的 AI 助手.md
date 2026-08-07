---
title: Git 自动化：构建一个帮你提交代码、管理分支的 AI 助手
feedId: 31975
source: 综合讨论
publishedAt: 2026-08-07
---

# Git 自动化：构建一个帮你提交代码、管理分支的 AI 助手

## 背景

如果你每天在终端里重复输入 `git add -A`、`git commit -m "fix bug"`，或者在功能分支命名上犹豫不决，迟早会觉得这些操作像某种机械劳动。更糟糕的是，临时切换上下文后，提交信息（commit message）往往写得随心所欲，等到追溯提交历史时才发现一片混乱。

AI 编程助手已经能帮我们写代码、修 bug，但 Git 操作本身却被大多数工具忽略。OpenClaw、MCP（Model Context Protocol）这类 Agent 框架的出现，让“让 AI 接管部分 Git 工作”这件事变得可行。这篇文章不讨论“全自动 Git 机器人”，而是给出一个务实的方案：**构建一个受控的 Git 自动化助手，负责生成规范的提交信息、执行安全的分支操作，同时把决策权保留在你手里。**

## 问题定义

我们需要解决的痛点很具体：

- **提交信息随意**：`fix`、`update`、`wip` 满天飞，团队很难维护 changelog。
- **分支命名混乱**：`feature1`、`test-branch`，几周后自己都不知道创建了什么。
- **重复操作多**：切分支、加文件、提交、推送，每次都要走一遍流程。

理想状态是：用户描述意图（比如“把 index.py 的改动提交一下，类型是修复缓存 bug”），Agent 自动分析变更、生成符合规范的 commit message、执行提交，必要时创建或切换分支。整个过程可预览、可干预，而不是黑箱操作。

## 实现方案：基于 OpenClaw 的 Git 工具链

下面给出一个可复现的轻量实现，核心依赖只有三个：OpenClaw Agent 框架、系统 Git CLI、一个可控的 Shell 执行工具。

### 1. 环境准备

- Python 3.10+
- 安装 OpenClaw：`pip install openclaw`（或其他 Agent 运行框架）
- 确保本地 Git 可用，且工作区处于正常状态

### 2. 定义安全的 Git 工具

Agent 的危险在于直接执行任意 shell 命令。我们需要封装一个“白名单工具”，只允许执行预定义的 Git 子命令，并过滤危险参数。下面的代码片段展示了核心思路：

```python
import subprocess
from openclaw.tools import tool

ALLOWED_COMMANDS = ["status", "diff", "add", "commit", "branch", "checkout", "log"]

@tool(
    name="run_git",
    description="执行受控的 Git 命令。参数为命令列表，如 ['diff', '--cached']。"
)
def run_git(args: list[str]) -> str:
    if not args or args[0] not in ALLOWED_COMMANDS:
        return "Error: 不允许执行的 Git 命令。"
    try:
        result = subprocess.run(
            ["git"] + args,
            capture_output=True,
            text=True,
            timeout=15,
            check=False
        )
        return result.stdout + result.stderr
    except Exception as e:
        return f"Git 执行异常: {str(e)}"
```

注意我们没开放 `push`、`reset --hard` 等危险操作，推送需用户手动完成。

### 3. 构建 Agent 的 Prompt

Agent 需要学会“读懂 diff 并输出约定式提交（Conventional Commits）”。典型的系统提示词可以这么写：

```
你是一个 Git 自动化助手。用户会要求你提交代码，你需要：
1. 先用 `run_git status` 查看工作区状态；
2. 用 `run_git diff` 或 `run_git diff --cached` 查看变更；
3. 总结变更，生成一条符合 Conventional Commits 规范的提交信息，格式：
   type(scope): description
   类型包括：feat, fix, refactor, docs, test, chore 等
4. 将提交信息展示给用户，用户确认后再执行 `run_git add` 和 `run_git commit -m "<信息>"`。

对于分支管理，当用户说“开始新功能 <feature>”时，从 main 或 master 创建名为 feature/<feature> 的分支。
```

### 4. 典型工作流

以“提交当前所有变更”为例，对话过程大约是这样：

```
用户: 提交这次的修改，类型是 fix，修复了用户登录缓存没有失效的问题。

Agent:
[调用 run_git status] → 输出 modified: auth.py, utils/cache.py
[调用 run_git diff] → 返回 diff 内容（省略）
分析后生成：fix(auth): 修复登录后缓存未失效导致旧会话可用的问题
并提示用户确认。

用户: 确认。

Agent 依次执行：
run_git add auth.py utils/cache.py
run_git commit -m "fix(auth): 修复登录后缓存未失效导致旧会话可用的问题"
返回提交成功。
```

创建功能分支时：

```
用户: 基于 main 创建一个支付模块的 feature 分支。

Agent:
run_git checkout main
run_git branch feature/payment-module
run_git checkout feature/payment-module
提示切换到新分支 feature/payment-module。
```

整个过程透明，Agent 只是充当“翻译 + 执行者”。

## 踩坑点与排障指南

实际集成时会遇到几个典型坑：

**1. diff 内容过长导致上下文溢出**
大型 PR 的 diff 可能超过模型上下文窗口。解决方法：让 Agent 先 `status` 查看变更文件列表，然后逐个文件 diff，或只 diff 处理逻辑相关的文件。也可以设置 token 截断，但会丢失细节。工程化做法是：对于超过 200 行的 diff，要求 Agent 生成摘要而不是逐行分析，然后人工审查。

**2. 模型生成的提交信息不符合规范**
即使提示词写得很清楚，模型有时会在 type 后加标点、混用中英文。可以通过代码后处理进行格式校验，若不合法则让模型重新生成，最多重试 2 次。例如使用正则检查 `^(feat|fix|refactor|docs|test|chore)(\(.+\))?: .+`。

**3. 并发处理与状态污染**
如果用户手动修改了文件，同时 Agent 正在执行 `git add`，可能出现竞争。Agent 应在每次操作前重新执行 `status`，如果发现工作区已变更就暂停并通知用户。

**4. 本地钩子（pre-commit）导致的提交失败**
如果项目配置了 pre-commit hooks，提交可能因代码格式问题失败。Agent 必须捕获 `run_git commit` 的 stderr，并把错误信息完整返回给用户，而不是假装提交成功。

**5. 分支操作的授权**
自动切分支不会破坏历史，但自动创建分支可能引入混乱的命名。建议在 Agent 中预设分支命名规则（如 `feature/`, `fix/`, `chore/` 前缀），不合规的请求直接拒绝。

## 可复用建议

- **优先使用 MCP 社区方案**：如果 OpenClaw 环境支持 MCP，可直接接入官方的 Git MCP 服务器，避免自己实现工具函数，安全性和稳定性更好。
- **做成可插拔 Skill**：将这个 Git 助手封装成 OpenClaw 的 Skill 或插件，可以在不同项目中复用，只需调整提交规范模板。
- **与 CI / pre-push 联动**：可以在 Agent 工作流中加入一步：生成 `CHANGELOG.md` 条目，并提醒用户推送后创建 PR。
- **保持人类最终控制**：永远不要让 Agent 直接执行 `push`、`rebase`、`reset --hard`，所有高危操作设置确认门槛。自动化不是无人化。

## 总结

用 AI 管理 Git 提交和分支，不是为了替代开发者的判断，而是为了消除琐碎、提高一致性。一套合理的“受控自动化”方案，只比手动多了一层生成与确认，却让提交历史变得规范可读，分支管理不再随意。试试把 `git commit -m "update"` 的坏习惯交给助手纠正，你自己专注于更重要的设计与逻辑。

如果你已经在 OpenClaw 或类似 Agent 框架上实践过，欢迎补充你的安全加固方案或分支管理策略。

---

