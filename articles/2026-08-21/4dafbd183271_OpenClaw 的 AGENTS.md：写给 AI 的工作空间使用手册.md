---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 33954
source: 综合讨论
publishedAt: 2026-08-21
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景

在 OpenClaw 里，Agent 经常要跨文件修改、调用 MCP 工具、跑脚本、查数据目录。刚开始工作空间很简单，上下文靠人肉说明还能应付。但当工作空间里塞进多个 MCP server、插件、脚本目录、数据目录后，Agent 很容易“迷路”。

AGENTS.md 就是一个放在工作空间根目录、专门给 AI 读的使用手册。它不替代 OpenClaw 的配置，也不替代 MCP 定义，而是告诉 Agent：这个工作空间里有什么、哪些事可以做、哪些事不要碰、常见任务怎么执行。

## 问题：没有 AGENTS.md 时发生了什么

实际使用中，没有 AGENTS.md 的典型故障包括：

- Agent 找不到脚本，开始自己“发明”命令；
- 误改 `.openclaw/` 或 MCP 配置文件，把环境搞坏；
- 调用 MCP 工具时参数不对，反复试错；
- 每次新会话都要重新解释目录结构和约定，消耗 token 和耐心；
- 生成文件直接扔在根目录，工作空间越来越乱。

这些问题不是模型能力不够，而是缺少稳定的工作空间级上下文。

## 做法：一个最小可用的 AGENTS.md

OpenClaw 支持把 `AGENTS.md` 放在工作空间根目录，作为 Agent 的上下文入口。如果你的版本没有自动加载，可以在启动配置或 system prompt 里显式引用该文件。

下面是一个工程化程度比较高的最小模板：

```markdown
# AGENTS.md

## Workspace Map
- `scripts/`：可执行脚本，通过 `make` 或 `task` 调用
- `data/`：输入与原始数据，只读，不要原地修改
- `output/`：所有生成物统一放这里
- `.openclaw/`：OpenClaw 配置，非明确要求不要改

## MCP & Plugins
- `filesystem` MCP：仅允许访问 `./workspace` 与 `./data`
- `github` MCP：只读，提交前必须人工确认
- `pdf` 插件：处理 PDF 前先确认文件存在

## Commands
- 运行测试：`pytest tests/ -q`
- 构建：`make build`
- 环境自检：`openclaw doctor`

## Conventions
- 新脚本统一放 `scripts/`，命名用 snake_case
- 输出文件必须放 `output/`，不要污染根目录
- 修改配置前先备份，并在回复里说明 diff

## Guardrails
- 禁止删除 `.git/`、`.openclaw/`
- 禁止直接执行 `rm -rf`；清理前先列出目标文件
- 长时间网络任务先询问，不要自动挂起
```

## 步骤

1. **审计工作空间**：先列出目录、MCP server、插件、常用命令和已有约定。
2. **写最小版本**：不要追求完整，先覆盖“环境—工具—命令—红线”四块。
3. **放到根目录**：确认为 `AGENTS.md`，编码 UTF-8。
4. **用真实任务测试**：让 Agent 执行一个跨目录任务，观察它是否按规则走。
5. **纳入版本管理**：和代码一起提交，变更走 review。

## 踩坑点

- **写得太长**：AGENTS.md 超过两屏后，Agent 容易忽略关键红线。保持精炼，细节可以链到别的文档。
- **只写命令，不写副作用**：比如 `make clean` 会删除什么、MCP 工具会改哪些文件，必须写清。
- **工具能力脱节**：AGENTS.md 里写的 MCP 工具名、参数名和实际配置不一致，Agent 会先相信手册再碰壁。
- **放敏感信息**：密钥、token、内网地址不要写进去。AGENTS.md 可能被 Agent 读取并带进对话上下文。
- **路径硬编码**：尽量用工作空间相对路径，不要写本机绝对路径。
- **写完不更新**：新加插件或调整目录后不同步，手册很快失效。

## 可复用建议

把 AGENTS.md 当成“运行手册”，而不是散文。每一条都应该是可执行、可验证的声明。

一个比较稳的结构是：

- Workspace Map：目录用途
- MCP & Plugins：每个工具写明“何时用、怎么调、副作用”
- Commands：常用命令与验证方式
- Conventions：命名、路径、提交习惯
- Guardrails：禁止事项和例外流程

另外，建议在 AGENTS.md 里加一段“如何验证环境正常”，例如先跑 `openclaw doctor` 或一个小测试命令。这样 Agent 遇到异常时有路径可循，而不是继续硬试。

## 总结

AGENTS.md 的价值不在于多聪明，而在于把工作空间知识从人脑和临时对话里，沉淀成 Agent 每次都能看到的稳定上下文。它成本很低，但能显著减少误操作和重复解释。对玩 MCP、插件和自动化的 OpenClaw 用户来说，这是比堆更多工具更先该做的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ee7508944d241b1f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/82ac7bca99e66225.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/393405ad449d0abd.png)

