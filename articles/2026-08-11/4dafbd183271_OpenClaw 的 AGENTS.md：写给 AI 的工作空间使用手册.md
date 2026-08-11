---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 32551
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：Agent 也需要一份“项目说明书”

OpenClaw 在多智能体、MCP 工具、插件编排上给了我们很大的灵活性，但灵活性同时也意味着“不确定性”。同一个 Agent，在 A 项目里表现稳定，换到 B 项目就反复出错——并不是模型变笨了，而是它缺失了人类开发者长期积累的那层隐形上下文。

你会因为熟悉项目而自然地使用 `src/services/` 下的某个模块，但 Agent 不知道文件结构；你会下意识地在调用外部 API 前先查接口文档，但 Agent 可能直接凭记忆编造参数。更常见的是，Agent 在一个失败策略上死循环，或者用错了 MCP 服务器、重复创建你已经明确过的文件夹结构。

这背后的本质问题是：**项目工作空间中存在大量“只有人知道、AI 完全不可见”的约定和约束。**  
OpenClaw 的 `AGENTS.md` 就是用来把这份隐性知识写下来，注入到 Agent 每次决策的上下文里，让它不再瞎猜。

## 问题：没有约束的 Agent 有多容易失控

举几个真实场景：

- 一个 Node.js 项目里，Agent 被要求添加一个新的 CLI 命令，它没有去读 `package.json` 里的 `bin` 配置，而是直接在根目录新建了一个 `.js` 文件，并且用 `require` 引入了不存在的模块。
- 另一个项目同时挂载了多个 MCP 服务器（Filesystem、GitHub、Postgres），Agent 在查询数据库时错误地调用了 Filesystem 工具去读取本地 dump 文件，浪费了巨量 token。
- 团队约定 API 路由统一写在 `src/routes/v2/` 下面，但 Agent 连续三次都把新路由放到了 `src/api/` 目录，原因是它记住了某个早期随机生成的目录名。

这些问题不是模型的错，而是我们没有给它提供明确的工作空间规则。`AGENTS.md` 就是解决这一层问题的第一道防线。

## 做法：如何编写并挂载 AGENTS.md

### 1. 创建文件
在项目根目录下直接新建 `AGENTS.md`（文件名可配置，但默认约定如此）。OpenClaw 从 v0.9 开始支持自动检测并加载该文件，如果版本较低，也可以在 agent 配置里显式引入，例如：

```json
{
  "instructions": [
    { "type": "file", "path": "AGENTS.md" },
    { "type": "text", "content": "You are an expert developer..." }
  ]
}
```

建议始终使用相对路径，并确保文件编码为 UTF-8、无 BOM，避免解析异常。

### 2. 内容结构
一份可用的 `AGENTS.md` 不需要写成文档，而应该像一份“操作手册”，用模型最能理解的简洁英文编写（英文指令准确度通常远高于中文）。推荐的最小结构：

- **Project Overview**：一句话说明项目用途，技术栈。
- **Directory Structure**：列出关键路径及各自职责，例如 `src/commands/` 放 CLI 命令、`src/mcp-tools/` 放自定义 MCP 服务等。
- **Tool & Plugin Usage**：明确每个 MCP 服务器或插件的使用时机和禁止行为。比如：“`github` MCP server can be used for creating issues, but NEVER for reading source code.”
- **Constraints**：硬性禁止项，如“所有新增文件都必须放在 `src/` 下”、“不要直接修改 `dist/`”。
- **Workflow Hints**：给出典型任务的首选执行步骤。例如：“When adding a new API route, first read `src/routes/index.ts` to understand pattern, then create file in `src/routes/v2/`.”

重要规则放在文件前面，因为 Agent 的上下文窗口有限，后面的内容可能被截断或注意力稀释。

### 3. 验证与迭代
写好初版后，用一个非破坏性任务测试：比如让 Agent “列出当前项目支持的所有命令行子命令，并说明它们的文件位置”。观察 Agent 是否正确使用了你规定的目录和工具，再根据错误行为回头修改 `AGENTS.md` 的措辞，增加更明确的否定示例。

## 踩坑点：这些细节会坑到你

- **路径解释歧义**  
  你写的 `components/Button` 在 Agent 眼里可能相对于它当前所在子目录。解决方案是在 `AGENTS.md` 开头就声明：“All paths in this file are relative to the project root.” 同时要明确告诉 Agent，执行命令时必须先 `cd` 到项目根目录。

- **与内置行为冲突**  
  某些 OpenClaw 版本有默认的系统提示，其中的工具使用偏好可能与你的 `AGENTS.md` 冲突。建议先用一次不带 `AGENTS.md` 的简单对话，观察 Agent 的默认行为，再针对性覆盖，而不是盲目写一堆“应该用这个、不应该用那个”。

- **内容过载和 Token 浪费**  
  有的人把 API 文档、代码规范、甚至整个 README 都塞进去，结果 Agent 的关键约束在第一轮就滚出了上下文。经验值是尽量控制在 200 行以内，动态信息用 MCP 工具实时查询，而不是写在静态文件里。

- **忽略环境差异**  
  本地用的是 MCP 的 filesystem 服务，CI 环境可能没有挂载。如果 `AGENTS.md` 里强制使用某个不在所有环境都存在的工具，Agent 会在缺失工具时反复报错。可以为不同环境准备不同版本，或者用占位符和条件提示来软化约束。

## 可复用建议

1. **模板化复用**  
   团队内共享一个基准 `AGENTS.md` 模板，包含目录结构说明、常用 MCP 工具清单、禁止操作示例。每个项目 Fork 后只需修改项目特定部分，大幅降低编写成本。

2. **与 MCP 配置显式联动**  
   在 `AGENTS.md` 中不要只说“使用 Postgres 工具”，而要写成：“Use `postgres` MCP server for all database queries. Do not use `filesystem` tool to read SQL files.” 这类两两对比的规则最能让 Agent 理解边界。

3. **版本管理与评审**  
   `AGENTS.md` 是项目工程的组成部分，应该随代码一起进入版本控制。任何修改都应由人审阅，避免 AI 自动生成的规则残留导致行为退化。你甚至可以把“更新 AGENTS.md”作为 OpenClaw 任务链的一个标准步骤。

4. **避免变成“提示词炼金术”现场**  
   `AGENTS.md` 越是简洁、直接，越不容易失效。不要用祈使句循环同一个意思，一个规则说一遍就够。Agent 不是靠重复来控制，而是靠精确的否定与边界描述。

## 总结

`AGENTS.md` 是 OpenClaw 项目中成本极低、见效很快的一种约束手段。它不会让 Agent 变聪明，但能帮它变“懂事”——知道哪些坑不能踩，哪条路是这个项目约定俗成的走法。在一个多插件、多 MCP 工具的自动化环境里，把隐性知识做显性化，是让 Agent 真正参与协作而非制造噪音的第一步。

如果你已经在用 OpenClaw 做复杂工作流，不妨花 15 分钟写一份 `AGENTS.md`，然后对比前后 Agent 的行为。你会发现，一个原本需要反复纠正的代理，突然变得像团队里读过 README 的新同事。

---

