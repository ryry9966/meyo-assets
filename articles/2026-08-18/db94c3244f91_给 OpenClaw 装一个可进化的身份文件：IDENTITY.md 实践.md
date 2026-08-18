---
title: 给 OpenClaw 装一个可进化的身份文件：IDENTITY.md 实践
feedId: 33731
source: 综合讨论
publishedAt: 2026-08-18
---

# 背景

在 OpenClaw 上接 MCP、插件、自动化任务时，很多人会把行为约束临时塞进 system prompt 或插件描述里。短期能用，但 agent 一旦重启、切换会话，或者多 agent 协作，约束就开始漂移：同一个任务今天这么做，明天又忘了；工具选择不稳定；踩过的坑每次都重新踩一遍。

IDENTITY.md 的思路很简单：把 AI 的身份、边界、规则、决策经验外置成一个文件，让 agent 每次启动都读，每次结束都能沉淀。它不是魔法，而是一份可版本化、可 review 的“项目记忆”。

# 问题

常见的痛点有三个：

1. **约束分散**：规则写在 prompt、插件配置、MCP 描述、脚本注释里，改一处漏一处。
2. **行为漂移**：会话无状态，AI 每次只能靠当前上下文决策，经验不延续。
3. **多人/多 agent 冲突**：不同人给 AI 的指令互相打架，最后 agent 选择最不靠谱的一条执行。

IDENTITY.md 解决的不是“让 AI 更像人”，而是“让 AI 的约束和决策可管理”。

# 做法/步骤

## 1. 初始化 IDENTITY.md

放在项目根目录或 agent 工作目录下，结构保持简单：

```markdown
# IDENTITY.md

## Core Identity
- 角色：OpenClaw 项目自动化助手
- 目标：执行文件整理、数据抓取、MCP 工具调用

## Runtime Rules
1. 所有写操作前先做 dry-run。
2. 文件路径使用绝对路径。
3. 不允许删除 .git 目录。

## Decision Log
- 2025-01-15：MCP 网络工具超时频繁，规则：超时设 10s，失败重试 1 次。
```

核心是三层：**Core Identity** 定义不变的身份；**Runtime Rules** 定义可变规则；**Decision Log** 记录可复用经验。

## 2. 让 OpenClaw 每次读取

在 OpenClaw 的入口 prompt、bootstrap 脚本或 agent 模板里加入固定两步：

- 任务开始前，先读取项目根目录下的 `IDENTITY.md`。
- 任务结束时，如果产生了可复用经验，用 `update_identity` 命令追加 Decision Log。

如果 OpenClaw 没有原生支持，可以用 MCP 的 `filesystem` 工具读取，或者在外层脚本里把文件内容拼进上下文。关键是让读取成为固定动作，而不是 AI 自己决定要不要读。

## 3. 用固定格式记录决策

Decision Log 不要写成流水账。每条只记四件事：

- **日期**
- **场景**：遇到什么问题
- **决策**：选择了什么方案
- **生效规则**：下次遇到类似情况该怎么做

例如：

```markdown
- 2025-01-16：批量重命名时误删了扩展名。决策：重命名前先列出变更清单。规则：任何批量重命名必须先输出变更预览。
```

这样 AI 下次读到的是“规则”，而不是一段故事。

## 4. 限制修改权限

不能让 AI 随便覆盖 IDENTITY.md。最好只允许通过 `update_identity` 命令追加，不允许直接编辑 Runtime Rules。人工确认后再把有效规则移入正式段落。

# 踩坑点

1. **文件膨胀**：IDENTITY.md 超过 200 行后，AI 注意力会被稀释，反而忽略关键约束。旧决策日志迁移到 `archive/identity-journal.md`，主文件只留最常用的规则。

2. **AI 改坏文件**：必须加门禁。只允许追加，不允许覆盖；修改 Runtime Rules 需要人工确认。否则 agent 可能在一次失误后把核心约束删掉。

3. **路径和编码问题**：OpenClaw 运行目录不同会导致找不到文件。用绝对路径或环境变量 `OPENCLAW_IDENTITY_PATH`。Windows 下注意 CRLF 换行，统一用 LF 避免解析异常。

4. **规则冲突**：多个规则打架时，AI 会犹豫或选错。在文件里明确优先级：`Core Identity > Runtime Rules > Decision Log`，并写清冲突时以哪条为准。

5. **Decision Log 变成生活流水账**：只记录“可复用规则”。一次性操作、临时调试过程不要写进去，否则文件会迅速失去价值。

# 可复用建议

- **三层结构**：Core Identity 少而稳定；Runtime Rules 经常更新；Decision Log 只追加，定期归档。
- **用 git 管理**：每次变更单独 commit，坏改动可以回滚，也能看到“AI 是怎么进化自己的规则的”。
- **配合 MCP 工具**：用 `filesystem` MCP 或 `memory` MCP 读写，避免依赖 OpenClaw 特定指令。这样即使换框架，IDENTITY.md 依然可用。
- **定期清理**：每周或每 N 次会话后 review 一次，合并重复规则，删除过时条目。
- **把读取写进模板**：不管什么任务，开始读 IDENTITY.md，结束追加 Decision Log，形成闭环。

# 总结

IDENTITY.md 不是给 AI 一个“灵魂”，而是给 OpenClaw 的插件和自动化组合一个可进化的项目记忆。它成本低、可版本化、可 review，适合那些需要长期运行、多 agent 协作、或行为容易漂移的场景。不要把它神化，它就是一份被认真维护的工程文件——但这一点，恰好是很多 AI agent 实践里最缺的东西。

---

