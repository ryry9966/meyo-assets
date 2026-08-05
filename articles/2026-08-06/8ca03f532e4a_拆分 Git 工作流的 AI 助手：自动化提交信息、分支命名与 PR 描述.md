---
title: 拆分 Git 工作流的 AI 助手：自动化提交信息、分支命名与 PR 描述
feedId: 31814
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么 Git 操作总在拖慢节奏

如果你在团队中维护过多个迭代中的特性分支，大概率经历过这样的场景：凌晨修完一个紧急缺陷，随手打了个 `git commit -m "fix"`；第二天另一个人从主干切分支，命名 `feature/xxx-optimize` 却对应两个不相关的 issue。几个月后回溯变更历史时，`git log` 成了猜谜游戏，`branch` 列表变成垃圾场。这些手工操作不仅消耗认知资源，也让代码协作的可追溯性大打折扣。

我的日常开发环境大量依赖自主搭建的 Agent 工具链，其中 OpenClaw 负责串联不同任务。几个迭代之前，我把 Git 中高频重复的部分抽象出来，交给一个本地运行的轻量 AI 助手。它并不会替你决定代码写什么，但会接管那些格式规范类的活儿——生成符合 Conventional Commits 的提交信息、从 issue 标题推导分支名、为 PR 草拟初始描述。三个月的使用下来，本地 Git 日志的规范性明显改善，分支命名基本达到“一眼能看出对应哪个需求”的水平。

## 问题拆解：哪些环节最该自动化

手动 Git 操作中，真正低价值且易出错的主要是三类：

1. **提交信息的编写**：多数开发者知道需要写清楚 what/why，但紧急修复或小改动时容易偷懒，事后需要花时间 `rebase` 修正。
2. **分支命名**：不同人对同一 feature 的命名习惯差异大，加上日期、缩写等个人风格，导致分支列表可读性差。
3. **PR 描述**：把多个 commit 的内容聚合为一段背景和变更说明，常常要翻看 diff 才能写出，不如让 AI 直接生成初稿后人工精炼。

针对这三个点，完全可以利用 Git hooks 或自定义子命令，接入一个大语言模型接口（本地或云端），把“遵守规范”这件事从强制执行变成自动填充。

## 实现一套可工作的原型

我选择用 Python 脚本 + OpenAI 兼容 API 来实现，模型使用本地部署的 7B 参数模型（为了低延迟和不依赖外网），也可以对接任何支持 Chat Completions 的服务。脚本存放在 `~/.git-ai-hooks/` 下，然后在各个项目的 `.git/hooks/` 里软链过去。

### 1. 提交信息自动生成

使用 `prepare-commit-msg` hook，在用户执行 `git commit` 且未给出 `-m` 时触发。核心逻辑：获取暂存区 diff，限制长度（避免大变更让模型超时），构造 prompt 让模型输出 `<type>(<scope>): <subject>` 格式的一行摘要，再加一个空行和可选的 `<body>`。

踩坑点：
- **diff 过大**：一次提交改动数千行时，diff 本身可能超过模型上下文窗口。解决方法是只取前 N 行，或先统计变更文件数量和行数，若超过阈值则只生成基于文件名和变更类型的简要提示，并提醒开发者手工填写。
- **语言混杂**：diff 中可能出现中文注释，导致模型输出的 commit message 混入中文。通过 prompt 明确要求“English only, no Chinese characters”，并后置过滤。
- **请求超时**：本地模型推理若超过 2 秒，commit 命令会感觉卡顿。选用量化后的小模型，并缓存重复的 prompt 前缀可以降低延迟。

脚本示例片段（精简）：

```python
import subprocess, sys, openai

DIFF = subprocess.getoutput("git diff --cached -- . ':!*.lock'")
if len(DIFF) > 4000:
    DIFF = DIFF[:4000] + "\n... (truncated)"

resp = openai.ChatCompletion.create(
    model="local-model",
    messages=[
        {"role": "system", "content": "Generate a conventional commit message (English only)."},
        {"role": "user", "content": f"Changes:\n{DIFF}"}
    ],
    temperature=0.3
)
msg = resp.choices[0].message.content.strip()
with open(sys.argv[1], "w") as f:
    f.write(msg)
```

将这个脚本保存为 `prepare-commit-msg`，并赋予可执行权限，然后在项目中做软链接：  
`ln -s ~/.git-ai-hooks/prepare-commit-msg .git/hooks/prepare-commit-msg`

### 2. 分支命名建议

我写了一个 Git 别名 `git ai-branch <issue-id>`，它会读取 GitHub/GitLab issue 的标题（通过 API），用模型生成一个符合 `type/issue-id-short-desc` 的 slug。例如输入 `issue-42`，输出 `feat/42-user-export-csv`。这一步没有使用 hook，因为分支创建时机不可控，用显式命令让开发者有控制感。

安全性注意：分支名只允许字母、数字和连字符，脚本内做了严格正则过滤，防止模型输出中包含空格或特殊符号导致命令失败。

### 3. PR 描述初稿

基于当前分支与目标分支的 `git log` 差异和文件变更统计，拼装 prompt 要求模型生成包含 Background、Changes、Testing Notes 三个段落的 Markdown 描述。使用 `post-push` hook 或手动 alias 触发。如果发现描述质量较低（例如模型胡说八道地编造测试步骤），后备方案是回退到只列举 commit 标题。

## 可复用建议

- **将 AI 挂钩变为团队共享配置**：把钩子脚本、prompt 模板和过滤规则放在一个独立 Git 仓库，通过 make 命令一键安装到本地。新成员加入时只需 clone 并执行 `make install-hooks`。
- **成本与延迟的平衡**：如果你习惯频繁 commit，建议使用本地模型；如果团队统一使用远程 API（如 GPT-3.5），可以设定每日调用上限，并对缓存 diff 哈希进行去重，避免重复请求。
- **不要完全依赖机器**：生成的提交信息与 PR 描述始终需要人工审核。现实是，模型偶尔会错误理解 diff，或把“删除旧功能”说成“添加新功能”。保留 `-m` 直接输入的能力，必要时覆盖。
- **与 OpenClaw 集成**：如果你已经在用 OpenClaw 担任日常任务 Agent，可以把上述脚本包装成一个工具模块，通过 Agent 对话主动询问“需要我根据当前 diff 生成 commit 吗？”——但我个人更习惯在终端直接感受反馈，可根据偏好选择。

## 总结

这套 AI 辅助 Git 工作流的本质不是“让 AI 替你编程”，而是用规则明确的模型输出去替代重复、易摇摆的人工格式决策。三个月以来，我自己的分支命名几乎不再出现含义不清的临时名称，`git log --oneline` 的可读性显著提高，而 PR 描述的草稿也省去了大量模板复制的时间。如果你也被 Git 中那些琐碎的规范化工作消耗着，不妨从一个小 hook 开始尝试，让助手接管形式，你专注于实质变更。

---

