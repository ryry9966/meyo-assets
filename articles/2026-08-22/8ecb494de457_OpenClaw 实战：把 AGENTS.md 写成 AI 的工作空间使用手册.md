---
title: OpenClaw 实战：把 AGENTS.md 写成 AI 的工作空间使用手册
feedId: 34101
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景与问题

OpenClaw 的 Agent 配置很容易只关注“能调哪些工具”，却忽略“在这个工作空间里应该怎么干活”。结果是 Agent 能读文件、能执行命令，但一上来先遍历全目录，误读 `.env`、误改 generated 文件、用错包管理器，甚至把 README 当操作说明。

README 是给人看的，不会告诉 AI 哪些目录能动、哪些命令是唯一入口、哪些文件是生成物。没有 AGENTS.md 时，Agent 每次都要重新猜测，上下文浪费在试错上，还容易触碰不该碰的配置。

AGENTS.md 解决的不是能力问题，而是边界和流程问题。

## 实践做法

在项目根目录或 `.openclaw/` 下放 `AGENTS.md`。里面写清楚四类信息：工作空间边界、标准命令、硬性规则、完成验证。下面是我在一个 TypeScript monorepo + MCP server 项目中实际使用的精简结构：

```md
# AGENTS.md

## Workspace boundary
- 可修改：packages/agent/src, packages/mcp-server/src, docs/
- 只读：infra/, generated/, .env*
- 禁止删除或移动 .openclaw/ 下的配置

## Commands
- 安装：pnpm install --frozen-lockfile
- 类型检查：pnpm --filter @repo/agent typecheck
- 测试：pnpm --filter @repo/agent test --run
- 构建：pnpm --filter @repo/mcp-server build

## Hard rules
- 不要直接编辑 packages/mcp-server/generated/protocol.ts
- 不要读取 .env 内容，只引用变量名
- 改动 src 后必须跑 typecheck 和 test

## MCP / plugins
- github MCP：只读 issue/PR，不创建内容
- filesystem MCP：只允许访问 ${WORKSPACE_DIR}/docs 和 ${WORKSPACE_DIR}/packages/agent/src
```

关键是写“验证方式”，而不是只写“跑测试”。例如：

> 测试命令必须输出 `pass` 或 `0 failed` 才算完成。

这样 Agent 才知道什么状态可以停止，而不是把测试失败也当成任务完成。

## 踩坑点

1. **文件过长被截断**  
Agent 上下文窗口有限，AGENTS.md 超过 200 行后，后面的关键规则可能被忽略。硬规则放最前，长示例放外链或折叠。

2. **相对路径导致 cwd 漂移**  
OpenClaw Agent 执行命令时 cwd 可能是工作区根、子目录或临时目录。规则里最好使用 `${WORKSPACE_DIR}`，或者从工作区根开始的路径，避免写 `./src` 这种依赖当前目录的写法。

3. **把 AGENTS.md 当沙箱**  
写了“禁止 `rm -rf`”并不代表权限层会拦截。OpenClaw 的工具白名单、allowed-tools、MCP allowlist 才是真正的执行边界。AGENTS.md 负责约束行为，权限层负责拦截危险动作，两者必须配合。

4. **和 README 重复**  
README 讲背景和架构，AGENTS.md 讲操作约束。不要把大段说明复制进来，否则 Agent 读了还是不知道怎么做。

5. **命令变更后不更新**  
AGENTS.md 里的命令过期后，Agent 会持续用旧命令失败。可以在 CI 里加一个检查，对比 `package.json` scripts 与 AGENTS.md 中出现的命令是否一致。

## 可复用建议

- **全局 + 项目两层**  
`~/.openclaw/AGENTS.md` 放通用规则，例如“不读密钥、先读项目 AGENTS.md”；项目根放项目级规则。避免每个项目重复写通用内容。

- **硬规则 / 软建议 / 示例三段式**  
硬规则保持 5–10 条，短句、可验证；软建议放工作偏好；长示例放外链或折叠区。

- **把完成标准写进规则**  
例如：  
  > 修改完成 = typecheck 通过 + 单测通过 + 无新增 lint warning。

- **新会话测试**  
每次更新后新开 OpenClaw 会话，让 Agent 先复述一遍关键规则，确认没有因为文件过长或位置问题而丢失。

## 总结

AGENTS.md 是给 AI 的工作空间使用手册，不是权限配置，也不是 README。它解决的是 Agent 在一个陌生仓库里“该做什么、不该做什么、做完怎么验证”的问题。写得好的标准只有一条：换一个全新会话，Agent 读完 AGENTS.md 后不需要再追问基础边界，也不会把时间浪费在试错上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4551c0a9b4aa09de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/56de207933894164.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b36c35810ddad281.png)

