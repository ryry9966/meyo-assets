---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 33377
source: 综合讨论
publishedAt: 2026-08-16
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景：Agent 的“身份漂移”

在 OpenClaw 里接入 Agent、MCP 工具和自动化流程后，容易出现一个现象：同一个 Agent，昨天还能严格按你的偏好输出，今天换了个会话就变得啰嗦、风格漂移，甚至忘记了项目中已经约定好的决策。

这不是模型能力的问题，而是**身份上下文没有被持久化**。每次新会话开始时，Agent 只能依赖零散的 system prompt、项目说明和临时注入的片段。这些内容往往互相矛盾、过时，或者根本没有定义“我是谁、我该怎么做、我不能做什么”。

IDENTITY.md 要解决的就是这件事：给 Agent 一个可读、可维护、可进化的身份文件，而不是每次都从零开始“猜”。

## 问题：为什么默认 system prompt 不够

只靠 system prompt 有几个明显缺陷：

- **不可积累**：每次对话结束，学到的东西不会自动沉淀回身份定义。
- **不可版本化**：改了 prompt 之后，很难回答“这个 Agent 上周的行为为什么和现在不一样”。
- **项目上下文缺失**：Agent 知道自己是助手，但不知道当前仓库的目录约定、禁用命令、发布流程。
- **边界模糊**：哪些 MCP 工具可以用，哪些只能读不能写，往往散落在不同配置里。

IDENTITY.md 的思路是：把身份从“一次性提示词”升级为“项目里的工程文件”。它像一个团队的 `CONTRIBUTING.md`，只不过读者是 AI。

## 做法：一个最小可用的 IDENTITY.md

在项目根目录或 `.openclaw/` 下创建 `IDENTITY.md`，内容不要写成散文，而是结构化块。我常用的最小结构如下：

```markdown
---
id: repo-helper
version: 7
updated: 2025-01-20T10:00:00+08:00
token_budget: 800
---

# Identity
你是这个仓库的工程助手，主要协助完成代码维护、脚本编写和文档整理。

# Goals
- 优先保持变更小而可回滚
- 输出前先确认是否影响构建
- 对不确定的依赖升级给出风险提示

# Boundaries
- 不要直接执行破坏性 git 操作
- MCP 文件工具只读，除非用户明确要求写入
- 不生成凭据、不读取 .env 的完整内容

# Workflow
1. 先查看 git status 和最近提交
2. 修改前说明影响范围
3. 完成后给出验证命令

# Memory
- 2025-01-18: 用户不喜欢自动添加 emoji，文档默认使用中文
- 2025-01-19: 构建失败通常是 Node 版本不匹配，先检查 .nvmrc

# Evolution Log
- v6 -> v7: 增加 MCP 文件工具只读边界
- v5 -> v6: 拆分 Memory 和 Evolution Log
```

这个文件的关键不是“全”，而是**有一致结构**。`frontmatter` 里的 `version`、`updated` 和 `token_budget` 可以帮你做版本管理和注入控制。

## 加载方式：不要全量塞进每次对话

在 OpenClaw 中，我建议至少有两种加载方式：

1. **冷启动注入**：会话开始时读取 `IDENTITY.md`，但只注入核心块，例如 Identity、Goals、Boundaries，控制在 500–800 token 内。
2. **按需读取**：把 Memory 和大段 Workflow 放到单独文件，通过 MCP 资源或脚本按需读取。例如 Agent 需要了解发布流程时，再调用工具读 `docs/release.md`，而不是一开始全量加载。

如果 OpenClaw 支持自定义启动指令或 MCP server，可以写一个很薄的加载器：解析 frontmatter，根据 `token_budget` 截取核心块，其余部分注册为可检索资源。

## 可进化：让 IDENTITY.md 自己更新

IDENTITY.md 如果只靠人工维护，很容易过时。我一般会加一个“演进”机制：

- 在会话结束或阶段性任务完成时，让 Agent 追加一条 `Memory`，只保留可复用的事实，而不是把整段对话塞进去。
- 用 git 管理 `IDENTITY.md`，每次更新都有 diff。你能清楚看到 Agent 为什么会改变行为。
- 定期做一次 consolidation：把超过 30 条的 Memory 合并、去重、归档，防止文件无限膨胀。

这样可以实现“可进化身份”：每次真实使用都会留下可追溯的痕迹。

## 踩坑点

1. **把 IDENTITY.md 当万能 system prompt 堆砌**
   文件超过 2000 token 后，核心身份反而被淹没。保持核心块精简，细节放外部资源。

2. **身份与项目绑定过深**
   如果 IDENTITY.md 里写满某个项目的路径和分支名，跨项目复用时会误导 Agent。项目相关内容应拆分到 `projects/<name>.md`，身份文件只保留通用边界和长期偏好。

3. **把敏感信息写进去**
   不要在 Memory 里记录 API key、完整 .env 内容或服务器密码。即使文件在本地，也可能被 Agent 误读或误传。

4. **只读不更新**
   没有更新机制的 IDENTITY.md 很快会变成“历史文档”。一定要让 Agent 有追加 Memory 的入口，哪怕只是简单的 slash command。

5. **忽略版本冲突**
   多人协作时，IDENTITY.md 的 Memory 块容易产生 git 冲突。建议把 Memory 拆成追加式条目，减少同一区域的重写。

## 可复用建议

- **拆分文件**：`IDENTITY.md` 只放核心身份；`MEMORY.md` 放事实；`PREFERENCES.md` 放风格偏好；项目细节放 `PROJECT.md`。
- **设置 token 预算**：核心注入不超过 800 token，其余按需读取。
- **用 frontmatter 做元数据**：版本、更新时间、适用范围，便于自动判断是否过期。
- **让更新可见**：所有变更走 git commit，commit message 写清楚“v6 -> v7: 限制 MCP 写权限”。
- **提供一个更新命令**：例如 `/identity update <事实>`，让用户和 Agent 都能快速追加。
- **定期压缩**：每 20–30 次更新后，人工或自动合并重复记忆。

## 总结

IDENTITY.md 不是一个炫技概念，而是 Agent 工程化中的基础设施。它解决的是长期使用中必然出现的三个问题：身份漂移、上下文不可复用、边界不可追溯。

在 OpenClaw 里，它的价值不在于文件本身，而在于你围绕它建立的加载、更新和版本管理流程。把 IDENTITY.md 当作一个**可进化的工程文件**，而不是一段更长的 prompt，才能让 Agent 在多次会话、多个项目之间保持稳定而不过时。

如果你正在搭建自己的 OpenClaw 工作区，不妨从 20 行开始：写清楚你是谁、边界在哪、怎么更新。然后让它随着使用自然生长。

---

