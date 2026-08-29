---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35306
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

AGENTS.md 不是另一份 README。README 是给人看的，AGENTS.md 是给 agent 看的工作空间运行说明。在 OpenClaw 里，agent 进入一个 workspace 后，需要尽快知道目录结构、命令边界、插件策略和哪些 MCP 工具能用。如果没有这份说明，每次都要在 prompt 里反复交代，而且 agent 一旦跑偏，代价通常不是写错文档，而是误改源码、误发消息、误写生产数据。

尤其在 MCP、插件和自动化脚本比较多的项目里，AGENTS.md 更像是给 agent 的“工作空间使用手册”。

## 常见问题

没有 AGENTS.md 或写得模糊时，常见翻车场景包括：

- agent 把源码目录当输出目录，生成一堆中间文件；
- 包管理器混用，npm/pnpm/yarn 来回切；
- 调用 MCP 工具时越权，例如直接操作远程仓库或发送群消息；
- 插件改了不该改的本地配置；
- 每次新会话都要重新解释一轮项目规则。

这些问题靠“更强模型”不一定能解决，靠一份清楚的 AGENTS.md 反而更稳定。

## 做法

### 1. 放对位置

我习惯在 OpenClaw workspace 根目录放一份 `AGENTS.md`。如果某个子模块规则差异很大，可以在子目录再放一份，并在根文件里说明优先级。

### 2. 内容尽量短，只写四块

我目前用的结构是：

```markdown
# AGENTS.md

## Layout
- src/ : read-only unless user asks
- out/ : generated artifacts, safe to write
- .local/ : temporary files only

## Commands
- package manager: pnpm only
- test: pnpm test -- --runInBand
- lint: pnpm lint
- build: pnpm build

## Forbidden
- never delete .env*
- never modify .github/ without asking
- never call mcp__github__merge unless user confirms

## Tool policy
- mcp__slack: read-only; do not post
- mcp__filesystem: allowed only under /workspace
- plugin: prettier: format only src/
```

这里的关键不是写得多全，而是每条都能直接执行。

### 3. 让 agent 先读后复述

在 OpenClaw 任务开头，我会要求 agent 先输出三条它认为最关键的约束。这样能很快判断它是否真的读了 AGENTS.md，而不是只扫一眼。

### 4. 纳入版本控制

AGENTS.md 跟代码一起提交。它不是临时提示词，而是 workspace 的一部分，应该和其他工程文件一样 review。

## 踩坑点

1. **写太长**  
   超过 200 行后，agent 很容易忽略后半部分。控制在 150–250 行以内比较合适。

2. **规则过时**  
   AGENTS.md 里记录的 MCP 工具或命令已经变了，但 agent 仍然坚持旧规则。这比没有规则更麻烦。改配置时务必同步更新。

3. **把密钥写进去**  
   AGENTS.md 可能被 agent 读取后传给外部 MCP。绝对不要写 token、密码、私钥。

4. **用模糊词**  
   “尽量不要”“建议避免”这类词对 agent 基本无效。要用 must、never、only 这种可执行约束。

5. **根目录和子目录规则冲突**  
   如果存在多份 AGENTS.md，必须在根文件里说明优先级，否则 agent 可能合并错误。

6. **让 agent 自己维护 AGENTS.md**  
   自动更新听起来很美，但 agent 可能把关键禁则改坏。如果要自动更新，至少先 dry-run diff，再人工确认。

## 可复用建议

- **模板化**：区分基础规则和项目特定规则，避免全局偏好污染单个 workspace。
- **加一条 smoke check**：在 AGENTS.md 里写清楚验证环境的方式，例如 `pnpm test -- smoke`。
- **给 MCP 工具标权限**：read-only、write、confirm 三类，减少误用。
- **记录维护日志**：文件底部写变更记录，方便排查“规则为什么变了”。
- **新 workspace 先测试**：让 agent 读取 AGENTS.md 并列出 5 条禁则，确认规则能被正确提取。

## 总结

AGENTS.md 本质上是给 agent 的工作空间契约。它不提升模型能力，只降低执行偏差。写短、写准、写禁区，比往 prompt 里堆更多背景更有效。对 OpenClaw、MCP、插件和自动化实践者来说，一份可执行的 AGENTS.md，往往比复杂提示词更有长期价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/e6294a26a09e9964.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7bbe59417116bf8b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/17e9b663a9e1c7cd.png)

