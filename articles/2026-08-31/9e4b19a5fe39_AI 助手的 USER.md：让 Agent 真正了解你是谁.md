---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 35462
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw、Claude Code 以及带 MCP 工具链的 Agent 实践中，项目级 `AGENTS.md`、`CLAUDE.md` 已经很常见。它们让 Agent 知道某个仓库的构建方式、测试命令、代码规范。但还有一种上下文经常缺位：**用户级上下文**。

结果是：Agent 知道这个仓库怎么跑测试，却不知道你的默认 shell、常用包管理器、工作目录边界，也不知道哪些自动化动作是你默认允许、哪些必须事先确认。于是每次会话都要重复解释，或者 Agent 自己猜。猜错一次，可能就是用错 Node 版本、跑到错误目录执行脚本，甚至触发你不想要的生产操作。

`USER.md` 就是解决这个问题的用户级说明书。它不替代项目规则，而是补充项目规则之外的“你是谁、你的环境是什么、你的默认约束是什么”。

## 问题

没有 `USER.md` 时，典型问题包括：

- Agent 不知道你惯用 `pnpm` 还是 `npm`，于是随机选择。
- 不知道你的工作目录边界，可能在 `~/.ssh`、`/etc` 或生产目录附近操作。
- 不知道哪些 MCP 工具可以默认调用，哪些必须二次确认。
- 每次都要重新描述“我在 macOS 上，用 zsh，Node 通过 fnm 管理”。
- 自动化脚本或插件触发时，Agent 因为缺少约束而执行不可逆动作。

这些问题不是模型能力不足，而是上下文没有结构化地告诉你。

## 做法与步骤

### 1. 确定文件位置和加载方式

建议放在用户级配置目录，例如：

```
~/.config/openclaw/user.md
```

如果你的 OpenClaw 使用 bootstrap 或 system prompt，可以加一行：

```
用户上下文文件：~/.config/openclaw/user.md
会话启动时读取一次。若与项目 AGENTS.md 冲突，以项目 AGENTS.md 为准。
```

不要把它注入为每条消息的前缀。只需要会话启动时加载一次，让它进入长期上下文即可。

### 2. 内容分块，只写操作事实

一个可用的 `USER.md` 可以长这样：

```markdown
# USER.md

## Identity
- 后端工程师，维护 macOS 和 Linux 双环境
- 工作目录：~/work
- 编辑器：neovim，不要自动格式化未配置插件的文件

## Environment
- 默认 shell：zsh
- Node：fnm 管理，当前主版本 22
- Python：uv
- Docker Compose：v2
- 本地镜像仓库：localhost:5000

## MCP / Tools
- 只读操作优先用 filesystem MCP
- git 历史查询用 git MCP
- shell MCP 禁止直接执行 rm -rf、git push --force、全局安装包

## Workflows
- 开始新仓库任务前，先读项目 AGENTS.md
- 不确定命令参数时，先 --help，再执行
- 同一任务连续失败两次，停下来确认环境

## Constraints
- 不得修改 ~/work 以外的文件，除非显式允许
- 不得提交任何密钥；使用 op 引用
- 所有变更尽量可逆：优先分支、cp -a、快照

## Recovery
- 如果上下文缺失或冲突，不要猜，直接提问
```

控制在 200 行以内。超过这个量级，Agent 的注意力会被稀释，关键约束反而失效。

### 3. 设置优先级

建议明确优先级：

```
项目 AGENTS.md > USER.md > 模型默认行为
```

项目规则离任务更近，应该优先。`USER.md` 负责补足项目没有说明的环境和个人约束。

## 踩坑点

1. **把 `USER.md` 当成提示词仓库**  
   塞入大量“你应该怎么做”的教程文本。Agent 需要的是事实，不是长篇建议。写“默认 shell 是 zsh”，不要写“请记住你是一个乐于助人的工程师”。

2. **写入密钥和令牌**  
   很多人为了省事，把 GitHub Token、数据库密码放进 `USER.md`，然后习惯性同步 dotfiles。这是高风险行为。安全信息应放环境变量或 secret manager，`USER.md` 只放引用方式。

3. **信息过期**  
   “我用 Node 18”写完后一年不更新，Agent 就会按旧环境执行。给它加日期或版本，或当成 dotfiles 定期 review。

4. **与项目规则冲突**  
   `USER.md` 写“总是用 pnpm”，但仓库锁文件是 npm。Agent 会犹豫或做错误选择。优先级必须写清：项目规则胜出。

5. **文件存在但未被读取**  
   只是放进 home 目录不等于 Agent 会读。必须在 OpenClaw 的 bootstrap、system prompt 或启动参数中引用它。

## 可复用建议

- 把 `USER.md` 当作 dotfiles 的一部分，放进私有版本库管理。
- 用短句、列表、关键值，避免散文段落。
- 建议增加“uncertainty rules”：信息缺失或冲突时，不许猜，先问。
- 多主机用户可拆分：`USER.mac.md`、`USER.linux.md`，或使用条件块。
- 团队共享仓库里不要提交个人 `USER.md`，可用 `USER.<name>.md` 并加入 `.gitignore`。
- 测试方式很简单：新开会话问 Agent“我的默认 shell、工作目录、禁止操作是什么”。如果答错，就说明加载或内容有问题。

## 总结

`USER.md` 不是魔法，也不是越长越好。它是一份小型用户操作剖面，让 Agent 在接触任务前就知道你的运行环境、工具偏好、行动边界和回退规则。对 OpenClaw/MCP/自动化用户来说，配合项目级规则，能显著减少重复说明，也减少 Agent 在边界上的猜测。

保持小、保持新、不存密钥、明确优先级。剩下的事，交给 Agent 去执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/fc420a5ba5a420a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/36155a6cea236e47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2d9d3ebb728ce45a.png)

