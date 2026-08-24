---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 34576
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw / Agent 的实践里，我们常把精力放在 MCP 工具链、插件权限、模型选择和自动化脚本上，却容易忽略一个更基础的问题：Agent 是否真的知道“你是谁、你要什么、边界在哪里”。

默认情况下，Agent 每次新会话都从零开始猜测。你想让它记住的偏好、常用的命令、不允许触碰的目录，通常只能靠每次手工补充。USER.md 就是解决这个问题的轻量入口。它是一份给 Agent 读取的用户上下文文件，让模型在对话开始前就带着你的背景、偏好和约束工作。

## 问题

实际使用中，USER.md 经常被写成两种极端。

一种是空泛的“我喜欢高效、清晰的输出”，Agent 读完不知道具体该怎么做。另一种是日记式流水账，把工作经历、技术栈、近期想法全塞进去。Agent 读完后被大量噪声干扰，关键任务中仍然可能误解你的意图。

安全问题也常见。有人把 token、Cookie、内网地址写进 USER.md，结果被同步到日志、被第三方插件读取，甚至被模型摘要后带出去。

另一个典型问题是作用域混乱。全局 USER.md 和项目级 AGENTS.md / CLAUDE.md 同时存在，规则互相覆盖，Agent 不知道听谁的。于是 USER.md 反而成了干扰源，而不是上下文基线。

## 做法与步骤

### 1. 确定文件位置与加载方式

OpenClaw 通常支持在用户目录或工作区放置 USER.md。建议全局一份放在 `~/.openclaw/USER.md` 或类似配置目录，用于跨项目偏好；项目级规则放项目根的 `AGENTS.md` 或 `CLAUDE.md`。

不要依赖模型“自动发现”。在系统提示词或启动配置里显式引用 USER.md 路径，并写清楚加载顺序。例如：先读全局 USER.md，再读项目级规则，项目规则冲突时以项目为准。

### 2. 控制内容结构

建议只保留五个模块：

- **身份与目标**：一两句话说明你的角色和主要工作流方向。
- **环境与边界**：操作系统、shell、包管理工具、允许操作的目录、是否允许联网或 sudo。
- **工具与插件偏好**：MCP 工具优先级、测试命令、日志查看方式、部署习惯。
- **沟通风格**：中文还是英文、先给方案还是直接改、错误时先诊断还是先回滚。
- **硬性约束**：禁止删除、禁止强推、禁止改数据库结构、敏感信息脱敏、不编造 API。

示例不要写太长，重点是“可执行”：

```markdown
## Identity
- Backend engineer, Python/FastAPI/PostgreSQL/Docker

## Environment
- OS: macOS, shell: zsh
- Allowed dirs: ~/work, /tmp/openclaw
- No sudo, no internal network access without confirmation

## Tool preferences
- Tests: pytest -q
- Logs: docker compose logs --tail=200
- MCP tools: filesystem, github, postgres

## Communication
- Respond in Chinese
- Show plan before execution
- Prefer minimal diffs

## Hard rules
- Never delete files or force push
- Never insert real secrets into output
- Ask before installing packages
```

### 3. 与 MCP / 插件协作

USER.md 不需要描述每个 MCP 工具怎么调用，而是写“当需要数据库结构时，使用 postgres MCP 的只读查询，不要直接改表”。

如果某个插件有特殊限制，写到约束区。例如“文件系统插件只能访问 `/workspace`，不允许读取 dotfiles”。这样 Agent 不仅知道工具存在，还知道你的使用边界。

### 4. 验证与迭代

写完后做几个典型任务：让 Agent 总结你的偏好；模拟一个需要权限判断的操作，看它是否先询问；检查它是否会把某段敏感信息带入输出。根据表现删掉无用的句子，保留真正影响行为的部分。

## 踩坑点

- **不要放密钥**。提交前搜索 `password`、`token`、`secret`、`api_key` 等关键词。
- **不要写太长**。USER.md 最好控制在 200–400 行以内，重点是可执行约束，而不是背景故事。
- **避免规则冲突**。如果项目 AGENTS.md 已经写了测试命令，USER.md 不要再重复，或者明确标注优先级。
- **注意编码和路径**。Windows 路径反斜杠、中文注释可能导致解析差异，建议统一 UTF-8，路径用正斜杠或环境变量。
- **不要幻想一次配置永久有效**。工作流会变，USER.md 需要版本管理和定期更新，最好放进 git 仓库一起维护。

## 可复用建议

- **分层管理**：全局 USER.md 存身份、沟通风格、安全边界；项目 AGENTS.md 存命令、目录、测试部署等具体规则。
- **模板化起步**：从简单模板开始，用注释标明“可删”的示例，不要一次追求完美。
- **写触发条件**：不要写“有些时候我需要 Docker”，而是写“当涉及本地服务联调时，优先用 docker compose 启动依赖”。
- **定期测试可执行性**：让 Agent 根据 USER.md 复述核心约束，发现偏差就修改原文。
- **做安全审计**：每次更改后检查是否有敏感信息、硬编码路径或过时规则。

## 总结

USER.md 不是提示词工程的花活，而是 Agent 长期稳定工作的基线。它让“个性化”从每次手工补充变成可复用配置，也让 MCP 工具和插件在更明确的边界内运行。

写 USER.md 的关键不是多，而是准；不是描述你的一切，而是让 Agent 在不知道答案时知道该问什么，在不需要问时直接按你的方式执行。对 OpenClaw 实践者来说，它值得被当成和插件配置一样的基础设施来维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e82268e6c26c5135.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e5a0db14c56a1090.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f613afb5064b6205.png)

