---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 34990
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

现在的 Agent 已经能读文件、调 MCP、执行命令、串联插件，但多数运行端缺少一个关键上下文：用户到底是谁。每次新会话都要重新自我介绍：我做什么、用什么环境、偏好什么风格、哪些目录不能碰。更麻烦的是，Agent 会按通用假设回答：默认你用 npm 而实际是 pnpm，默认生产环境可写而其实不该碰，默认你愿意让它跑迁移命令而其实只想先看方案。

USER.md 要解决的就是这件事：把“用户档案”从聊天记录里拿出来，变成 Agent 每次启动都能读取的稳定上下文。

## 问题

- 重复交代成本高，换一个会话就要重新解释；
- 输出与个人环境不匹配，命令、路径、包管理器全错；
- 越界风险：Agent 不知道你的红线，可能读错目录、改错配置；
- 上下文浪费：把个人背景当成聊天内容反复说，挤占了任务本身的 token。

## 做法 / 步骤

### 1. 选一个稳定位置

OpenClaw 通常支持在全局目录或项目 `.openclaw/` 下加载用户文件。建议全局放：

```text
~/.openclaw/users/<your-name>.md
```

项目级可选 `.openclaw/USER.md` 做覆盖。全局文件负责“你是谁”，项目文件负责“这个项目里你更在意什么”。

### 2. 写最小可用的 USER.md

不要写自传，只保留会影响 Agent 行为的差异点。一个可用模板：

```markdown
---
name: Alex
role: backend / platform engineer
os: macOS 14 + Ubuntu 22.04
stack: Python 3.12, FastAPI, PostgreSQL, Redis
shell: zsh
package_manager: uv, pnpm
editor: VS Code
constraints:
  - 不要访问 ~/.ssh
  - 不要修改 /etc 下配置
  - 生产环境操作前必须先给计划
preferences:
  - 方案优先，确认后再写代码
  - 默认给端口、环境变量和回滚方式
```

核心是“可执行”，不是“态度正确”。“默认使用 `uv sync && ruff check --fix`”比“请保持代码整洁”有用得多。

### 3. 注入方式

如果 OpenClaw 不原生支持 `USER.md`，可以把它追加到规则文件或 system prompt 末尾；也可以通过 MCP filesystem 让 Agent 在会话开始时读取一次。关键是让它在每次任务前都拿到这份上下文，而不是等你手动粘贴。

### 4. 允许 Agent 反写

加一个指令：“当我表达偏好变化时，更新 USER.md 的对应字段。”这样文件会随使用逐渐变准，而不是写完就过时。

## 踩坑点

- **太长**：USER.md 超过 500 字会挤占项目上下文。只保留差异点，不要重复“我喜欢高效”这类泛词。
- **放敏感信息**：不要把 token、密码、内网跳板机写进去。引到 secrets 工具或环境变量。
- **过时文件比没有更糟**：换 Python 版本、换包管理器后要同步。建议放进 dotfiles 仓库，每季度过一遍。
- **与项目规则冲突**：项目规范优先于个人偏好。可以在 USER.md 里写明：“当与项目规则冲突时，先指出冲突并等待确认。”
- **全局文件污染其他任务**：如果 USER.md 在全局目录，Agent 会在所有任务中加载，涉及他人项目时可能泄露个人偏好。可只对个人工作区启用，或用项目级文件覆盖。

## 可复用建议

- 分层维护：全局身份 + 项目覆盖，避免一个文件吃所有场景。
- 用 YAML frontmatter：结构清晰，Agent 更容易提取字段。
- 把“不知道就问”写进去：例如“环境不明确时先询问，不要按默认生成命令”。
- 设置边界字段：`forbidden_paths`、`default_commands`、`review_policy`。
- 小步开始：先写 5 个字段，跑一两天，发现重复解释时再加字段。

## 总结

USER.md 是成本很低但收益明显的上下文投资。它不是给 Agent 讲人生故事，而是提供可执行的差异信息：你是谁、你在哪运行、你用什么工具、你不能被碰什么、你希望怎么输出。保持简短、结构化、可维护，你的 Agent 会从“能执行”变成“懂你”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/0f20f5ba5b9cc835.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e6c316f12e84b356.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/479006dee6af7a16.png)

