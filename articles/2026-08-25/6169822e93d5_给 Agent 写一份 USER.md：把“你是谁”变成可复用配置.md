---
title: 给 Agent 写一份 USER.md：把“你是谁”变成可复用配置
feedId: 34635
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 这类 Agent 的自动化能力很强，但有一个常见短板：它对你的了解通常只停留在当前会话。下次新开任务时，它可能又回到“通用助手”状态，不知道你的操作系统、代码风格、常用命令、敏感边界。即便你每次都手动说明，上下文窗口也会被这些重复信息占据。

一个低成本的解法是维护一份 `USER.md`，把它作为 Agent 的持久用户上下文。它不替代项目里的 `AGENTS.md`，而是补充“谁来指挥、偏好是什么、边界在哪里”。

## 问题

实际使用中，没有 USER.md 或类似文件时，通常会遇到四类问题：

1. **冷启动损失**：每次新会话都要重新交代环境、工具链、路径偏好。
2. **风格漂移**：今天让它写 Python，明天写 shell，代码风格、命名习惯不稳定。
3. **越界风险**：Agent 可能读取你不想暴露的目录，或删除文件不确认。
4. **维护成本高**：为了让 Agent 稳定表现，需要在每个任务 prompt 里重复大量约束。

这些问题的根源不是 Agent 能力不够，而是缺少一个稳定、可版本化、可被 Agent 读取的“用户手册”。

## 做法 / 步骤

### 1. 确定文件位置和加载方式

推荐分两层：

- 全局：`~/.openclaw/USER.md`，描述跨项目、跨设备的用户级偏好。
- 项目级：`<project>/.openclaw/USER.md` 或项目已有的 `AGENTS.md` 中引用全局文件。

加载方式不要只靠手动粘贴。可以把 `USER.md` 挂到 Agent 的 startup context 或 system prompt 的固定位置，也可以在项目规则文件顶部写：

```text
Before any task, read ~/.openclaw/USER.md and follow it unless the project AGENTS.md overrides it.
```

如果你使用 MCP 文件系统工具，也可以让 Agent 在任务开始时先读取该文件并输出简短摘要，确认上下文已生效。

### 2. 内容结构化，而不是写散文

Agent 更容易遵守短句、列表、明确的“必须/禁止”。一个可用的最小模板如下：

```markdown
# USER.md

## 身份与环境
- OS: Debian 12
- Shell: zsh
- 时区: Asia/Shanghai
- 工作根目录: ~/projects

## 偏好与风格
- 交流语言: 中文，技术名词保留英文
- 代码缩进: 2 空格，单引号，不加分号
- 提交信息: `type(scope): subject`
- 解释策略: 先给结论，再给关键步骤

## 边界
- 只允许写 ~/projects 和 /tmp
- 删除、覆盖、git push 前必须确认
- 不读取 ~/.ssh、~/.aws、~/.config 下的敏感文件
- 所有密钥通过环境变量读取，不写入代码

## 常用工具链
- 包管理: pnpm
- 测试: vitest --run
- 构建: npm run build
- 部署: 手动确认后执行
```

### 3. 让 Agent 参与维护

不要只把 USER.md 当静态文件。可以给 Agent 一个规则：当发现你的新偏好或环境变化时，用文件编辑工具追加一条，而不是覆盖原有内容。例如：

```text
If you notice a repeated user preference, append a concise bullet to USER.md under the relevant section.
```

这样随着使用时间增加，文件会越来越贴合你的实际工作方式。

### 4. 与项目规则分层

当项目规则和全局 USER.md 冲突时，明确优先级。建议项目级 `AGENTS.md` 优先于全局 `USER.md`。如果项目有特殊构建命令或目录限制，应该在项目文件中写明覆盖项，而不是让 Agent 自行猜测。

## 踩坑点

1. **写了但没加载**：很多版本只在启动时读取上下文，修改 USER.md 后不会自动生效。改完建议新开会话，或手动让 Agent 重新读取。
2. **内容太泛**：如果 USER.md 写成“我喜欢高质量代码”，Agent 基本不会执行。要写具体到命令、路径、缩进宽度、是否确认。
3. **放敏感信息**：不要把 token、密钥写进 USER.md。路径可以用环境变量名代替。
4. **被 Agent 乱改**：Agent 可能把错误信息追加进 USER.md。建议用 git 追踪该文件，定期检查 diff。
5. **和项目文件冲突**：没有明确优先级时，Agent 可能同时遵守两套规则，行为不稳定。

## 可复用建议

- 把 USER.md 当成“入职文档”，而不是聊天记录。少写主观感受，多写可执行约束。
- 全局文件保持精简，项目级文件处理差异。
- 每月让 Agent 检查一次 USER.md 与实际使用是否一致，输出需要更新的条目。
- 如果你在团队中使用，可以维护一个团队版 `TEAM.md`，个人只覆盖差异部分。
- 给 USER.md 加版本号或 git 历史，避免误改无法回溯。

## 总结

USER.md 的核心价值不是“让 Agent 认识你”，而是减少重复沟通、降低越界风险、提升自动化任务的一致性。它不需要写得多长，需要的是结构化、可执行、有版本控制，并且真正被 Agent 在每次任务前读取。对于 OpenClaw 用户来说，这是低成本、高回报的工程化实践。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c53ddf7a383159d4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/fc347d0d2a129067.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/9b5d9e281e989354.png)

