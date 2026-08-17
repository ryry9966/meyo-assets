---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 33641
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景：为什么需要 USER.md

很多 OpenClaw 用户把精力放在 MCP 工具、插件和自动化流程上，但 Agent 仍然产出“正确但不对味”的结果。比如你习惯用 pnpm 而不是 npm，它总给你 npm 命令；你希望日志用英文，它偶尔输出中文；你明确说过不要在凌晨跑重任务，它还是把 cron 建议扔过来。

问题不在于模型能力，而在于它缺少关于“你”的稳定上下文。CLAUDE.md 解决“项目是什么”，USER.md 解决“你是谁”。在 OpenClaw 的 agent 工作区里，可以把 USER.md 作为常驻的用户档案，让每次任务启动都带上这些信息。

## 问题：Agent 的“失忆”从何而来

通常有三类：

1. 上下文窗口有限，历史对话被截断，早期偏好丢失。
2. 多任务/多项目切换时，用户身份和偏好没有跟随工作区走。
3. MCP/插件提供了大量工具说明，但不知道调用者是谁，只能按通用方式执行。

最典型的表现是：Agent 反复询问相同偏好、输出格式不一致、给出不符合你环境的命令。

## 做法：建立一份可用的 USER.md

我建议不要把 USER.md 写成自传。它应该是一份“给 Agent 的操作约束和偏好摘要”，控制在 200 行以内。

### 1. 放置位置

在 OpenClaw 工作区根目录放 `USER.md`，同时保留一份全局模板。如果支持全局记忆目录，可以在 `~/.openclaw/USER.md` 放通用信息；项目内只放项目相关偏好。

### 2. 内容结构

我的模板大致如下：

```markdown
# USER PROFILE

## Identity
- Role: backend developer / automation maintainer
- Timezone: Asia/Shanghai
- Language: Chinese for discussion, English for logs/commits

## Goals
- Reduce repetitive ops tasks
- Prefer observable, reversible automation

## Stack
- Node.js 20+, pnpm (not npm/yarn)
- Python 3.12, uv for dependency
- Docker Compose for local services

## Constraints
- Never run destructive commands without confirmation
- Do not suggest cron jobs between 01:00-06:00 local time
- Keep secrets in .env, never print them

## Workflow
- For bug reports: reproduce -> minimal case -> root cause -> fix
- For commits: conventional commits, no scope bloat

## Output Preferences
- Use Chinese for explanations
- Code comments in English
- Prefer tables for comparison, lists for steps
```

### 3. 让 OpenClaw 加载它

如果 OpenClaw 支持项目指令，可以在启动 prompt 中加入：

```
Read USER.md as persistent user context. If conflict with system prompt, follow USER.md only for style/preferences, not safety.
```

也可以用 MCP 文件系统工具让 Agent 自己读取，但更可靠的是作为系统/项目指令注入，避免每次消耗读取轮次。

### 4. 结合插件和 MCP

MCP 工具说明里如果有“用户偏好”，可以指向 USER.md 的 section，而不是复制内容。例如 Git 插件配置里写：

```
Commit style: see USER.md -> Output Preferences
```

这样避免多处维护同一偏好。

## 踩坑点

- **不要塞进所有历史**：USER.md 越大，越容易被忽略或稀释。只放高频、稳定、可执行的偏好。
- **不要放敏感信息**：API key、密码、内网拓扑不要进 USER.md。它是上下文，可能被发送到模型服务。
- **避免与系统指令冲突**：如果 USER.md 写“任何时候都先执行 X”，Agent 可能在危险操作上也照做。偏好不能覆盖安全边界。
- **过时信息比没有更糟**：换电脑、换技术栈、改变作息后，USER.md 会成为误导源。建议每周或每次环境变更时更新。
- **不要期望 100% 遵守**：模型可能遗漏，尤其 token 压力大时。对关键约束用更直接的任务指令确认。

## 可复用建议

1. 用“约束+偏好+工作流”三层结构，而不是流水账。
2. 项目相关偏好放项目内，跨项目身份放全局。
3. 在 USER.md 头部加 `Last updated: YYYY-MM-DD`，方便判断是否过时。
4. 将 USER.md 纳入 git 版本管理，变更可追溯；但全局文件不要提交到公开仓库。
5. 定期让 Agent 做“偏好审计”：把最近几次任务与 USER.md 对照，找出被忽略或已过时的条目。

## 总结

USER.md 不是让 Agent 更“懂你”的魔法，而是一份工程化的用户上下文契约。它的价值在于减少重复沟通、提高输出一致性、让自动化流程更贴合实际环境。对 OpenClaw 用户来说，它比多装几个插件更便宜，但回报往往更直接：少一次解释，少一次返工，少一条凌晨的 cron 建议。

---

