---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 34103
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景：Agent 缺的不是工具，是用户上下文

在 OpenClaw 或类似 Agent 实践里，我们经常花大量时间给助手接 MCP、写插件、调自动化流程，却忽略一件更基础的事：它并不了解“你是谁”。你的常用路径、命名习惯、提交规范、哪些命令不能自动执行，这些信息如果不显式告诉 Agent，它只能靠猜，或者每轮对话重新确认。

这类问题不是加更多工具能解决的。MCP 让 Agent 能访问文件系统、数据库、GitHub，但如果它不知道你的工作区边界和操作偏好，工具越多，误操作风险反而越大。USER.md 就是用来补齐这一层的轻量方案：一份机器可读的“用户说明书”，让 Agent 在进入任务前先读取你的上下文。

## 问题：为什么泛化输出和误操作反复出现

实践中常见几类问题：

- **上下文漂移**：你在 A 设备上告诉过 Agent 使用 zsh、工作区在 `~/dev/openclaw-test`，换到 B 设备或新会话后，这些信息全部丢失。
- **重复沟通成本**：每次都要重新说明“提交信息用 type(scope): summary”“文档用中文、术语保留英文”。
- **安全边界不清**：Agent 不知道 `~/.ssh`、`~/.aws` 不可读，也不知道 `rm -rf` 或 `git push --force` 不应该自动执行。
- **工具与插件约定缺失**：MCP server 命名、插件命令前缀、认证变量位置都是隐式知识，Agent 很容易调用错误。

这些问题不解决，Agent 就只能停留在“能跑但不够贴合”的水平。

## 做法：从一份最小 USER.md 开始

### 1. 选好位置和加载方式

在 OpenClaw 中，建议采用分层放置：

- 全局：`~/.openclaw/USER.md`，放跨项目通用偏好。
- 项目：工作区根目录的 `USER.md`，放项目特定约束。

加载方式要显式化。如果你的 OpenClaw 配置支持 file include，可以在主指令中写：

```text
Before any task, read USER.md if present.
```

否则可以在任务包装脚本中把 USER.md 注入上下文。也可以使用环境变量指定路径，方便多配置切换：

```bash
OPENCLAW_USER_FILE=~/.openclaw/USER.md
```

### 2. 内容分块

建议只保留 4-6 个 section，避免臃肿。一个可用的最小示例：

```markdown
# USER.md

## Environment
- OS: macOS 14 / Debian 12
- Shell: zsh
- Workspace: ~/dev/openclaw-test

## Preferences
- 回答语言：中文，术语保留英文
- 代码风格：4 空格缩进，优先函数式
- 提交信息：type(scope): summary

## Constraints
- 不要读取 ~/.ssh、~/.aws
- 不要自动执行 rm -rf、git push --force
- MCP 写操作前先确认

## Tooling
- MCP servers: filesystem, github, postgres
- 插件命令前缀: oc-plugin
```

### 3. 验证加载

新建一个会话，直接问：

> 我的工作区路径、提交规范、禁止操作是什么？

如果 Agent 能准确回答且来源是 USER.md，说明加载成功。如果回答泛化，优先检查文件路径、编码和主指令引用。

## 踩坑点

- **文件过长**：超过 150 行后，Agent 容易忽略尾部内容。控制在 80-150 行比较合适。
- **敏感信息**：不要写真实 token、密码或私钥。用变量引用如 `$GITHUB_TOKEN`，配合 secret 管理方案。
- **与 CLAUDE.md/AGENTS.md 冲突**：不同框架各有约定。若同时存在，明确优先级，例如 OpenClaw 只读 USER.md，项目规则放 PROJECT.md 并在 USER.md 中引用。
- **格式不被读取**：确保文件为 UTF-8，使用标准 Markdown heading，避免非常规符号。
- **多设备漂移**：USER.md 纳入 dotfiles 或私有 git 仓库，定期同步。注意不要将个人隐私提交到公开仓库。
- **模型过度依赖**：有些模型会把 USER.md 当成硬规则，忽略当前任务的特殊指令。可以在文件开头声明：“以下为默认偏好，冲突时以当前任务指令为准”。

## 可复用建议

- **分层管理**：全局 USER.md 放通用偏好，项目 USER.md 放项目特定，临时任务单独用 TASK.md。
- **模板化**：建立一个最小模板，新项目复制后 30 秒填完。
- **动态注入**：用脚本生成 USER.md 片段，例如将当前 git branch、最近提交、环境状态写入，让 Agent 每次都有最新上下文。
- **定期修剪**：每月检查，删除过时信息，保持文件精简。
- **固定测试**：新建会话后用固定问题验证读取效果，必要时调整位置和结构。

## 总结

USER.md 是 Agent 上下文工程的最小单元。它不替代 MCP、插件或自动化流程，但能显著降低沟通成本、减少误操作、提升输出一致性。

关键是克制：只写真正影响决策的信息，保持结构清晰，设定边界。可以先从 5 个 section 开始，验证加载后再逐步扩展。对于长期使用 OpenClaw 或类似 Agent 的用户来说，这比反复给 Agent 接新工具更务实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/67fc54bc897d4f5e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/afd02b16656999e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e6fcd9184f92bf47.png)

