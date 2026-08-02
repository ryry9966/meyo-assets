---
title: AGENTS.md：给 OpenClaw 工作空间写一份 AI 能读懂的操作手册
feedId: 31316
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：当 Agent 不再是单文件脚本

OpenClaw 的代理体系建立在多个 MCP 服务器、插件管道和自定义工具之上，工作空间很快就不再是一个扁平的脚本集合。你可能会同时面对本地文件系统操作、远程 API 调用、第三方 MCP 工具和团队约定的输出格式，而这些上下文没法每次都塞进 prompt 里，更不可能靠记忆传递。

多次被问到同一个问题时我意识到，Agent 缺的不是智能，而是一份稳定、可版本化的工作空间说明书。这就是 `AGENTS.md` 的作用——它不是给人类看的文档，而是直接注入给 AI 的运行时上下文。

## 问题：隐式约定带来的工程噪声

在没有 `AGENTS.md` 之前，常见的做法是通过 `systemPrompt` 或对话第一轮传递规则，但会迅速失控：

* **上下文漂移**：不同协作的代理实例拿到不同版本的规则，行为不一致。
* **Token 浪费**：每次会话都要解释“这个文件放在哪里”“为什么用这个 MCP 工具而不是那个”，长重复内容吃掉大量上下文窗口。
* **工具误用**：Agent 不清楚某个 MCP 服务器只能处理特定格式的输入，反复调用直到被限流或报错。
* **约束不可测**：规则散落在人脑和聊天记录里，新成员或新 Agent 无法快速对齐。

工程上需要一个像 `.cursorrules` 或 `.editorconfig` 一样的入口，专门描述“这个工作空间对 AI 意味着什么”。

## 做法：用 AGENTS.md 建立 AI 运行环境描述

`AGENTS.md` 放置在项目根目录，OpenClaw 代理启动时会读取该文件，作为系统提示的一部分接在标准提示之后。下面是一个经过实践精简的结构。

### 1. 文件定位
第一段直接告诉 AI 这份文件是什么、用于哪些 Agent 实例，以及它的优先级：
```markdown
# AGENTS.md — Workspace context for agent instances

This document is loaded as part of the system prompt for all agents
operating in this workspace. It defines conventions, tool usage policies
and expected output formats. Treat it as the single source of truth for
workspace-level rules.
```

### 2. 工作空间拓扑
描述目录结构和关键文件的作用，而不是罗列完整树：
```markdown
## Workspace layout

* `src/tools/` — MCP tool implementations, one file per tool.
* `data/` — local file system data used by tools; agents may write here.
* `output/` — all generated artifacts must be placed here.
* `config/mcp_servers.json` — MCP server registry. Agents must load
  tools from here, never hardcode endpoints.
```

这一节帮助 Agent 理解“去哪里找”而不是“怎么找到”。

### 3. MCP 工具使用策略
不是所有 MCP 服务器都对当前任务适用，需要白名单或提示：
```markdown
## MCP tool policy

* `file-system` — permitted for read/write under `data/` and `output/` only.
* `slack` — available for async notifications, never for real-time chat.
* `github` — read-only by default; write actions require explicit per-task approval.
* All other MCP servers listed in `config/mcp_servers.json` are disabled
  unless explicitly enabled in the task prompt.
```

这样可以将权限边界和工具治理写进上下文，减少越权调用。

### 4. 输出规范与质量约定
如果你要求 Agent 返回特定结构，比如 JSON Schema、Markdown 模板，写在这里：
```markdown
## Output conventions

* Summary results MUST be placed in `output/<task_id>.md` using the
  template defined in `templates/report.md`.
* When returning structured data, always include a `source_tool` field
  indicating which MCP tool produced the result.
* Do not invent new output directories; stick to `output/`.
```

### 5. 约束与禁忌
明确列出已被证明会出错的用法：
```markdown
## Constraints

* Do not modify files outside `data/` or `output/`.
* Avoid calling the same web search tool more than 5 times per task.
* Do not assume local timezone; always use UTC in logs.
```

### 6. 更新机制
文件末尾提醒人类维护者和 Agent 自身：
```markdown
## Maintenance

This file is version controlled. After updating MCP tools or workspace
structure, regenerate the relevant sections and commit alongside the
changes. Agents reading stale context will produce inconsistent results.
```

完整示例通常控制在 80-120 行，过长会导致上下文挤占工具调用所需的空间。

## 踩坑记录

1. **AGENTS.md 被当作一次性文档**：初期写完就不再更新，两周后引入新的 MCP 服务器，Agent 仍然认为它被禁用，反复拒绝调用。把它和 `package.json` 同等对待，改结构时必须同步更新。
2. **自然语言太啰嗦**：早期版本用了大段描述解释“为什么目录这样设计”，结果浪费 token 且 AI 不关心背景。改成命令式、陈述式语句后效果更好，每条规则一行。
3. **错误地把它当成安全边界**：`AGENTS.md` 只是提示，不是硬沙盒。真正敏感操作的权限控制必须在 MCP 服务器层面做，文件中写“禁止删除文件”只起到提醒作用，不能依赖。
4. **跨平台路径问题**：在 Windows 开发环境写的路径用 `\`，导致 Linux 容器里的 Agent 解析失败。统一使用正斜杠并测试。

## 可复用建议

* **结构分层**：分“拓扑—工具策略—输出—约束”四个核心节，避免碎点堆砌。
* **保持一次性上下文在别处**：具体任务参数、动态数据放任务 prompt，静态约定放 AGENTS.md。
* **用 Git 钩子检查一致性**：可写一个简单脚本在 pre-commit 时检查文件中列出的目录和配置是否存在，防止腐化。
* **为不同环境提供模板**：如果同一套代码要部署到多个工作空间（开发、测试、生产），用 `AGENTS.dev.md`、`AGENTS.prod.md` 并在启动参数中加载对应文件。

## 总结

AGENTS.md 解决的不是技术难题，而是工程上最常见的对齐问题：让 AI 和团队共享同一份工作空间的“肌肉记忆”。在 OpenClaw 这类多工具、多 MCP 的环境里，它比额外写一堆校验代码要轻量得多，而且天然可版本化、可评审。把它当成代码的一部分，而不是文档，可能是最务实的态度。

---

