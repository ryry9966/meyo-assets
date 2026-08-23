---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 34334
source: 综合讨论
publishedAt: 2026-08-23
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景：Agent 进了工作区，却在靠猜干活

在 OpenClaw 里接 MCP、写自动化脚本，或者让 Agent 直接参与项目改动时，有一个问题会反复出现：Agent 第一次进入工作区时，并不了解这个项目的真实约定。它会用全局知识硬套，把依赖装错位置，在错误目录跑命令，忽略项目里已有的 Makefile、脚本或本地 CLI，甚至无视端口、环境变量和文件边界。

更麻烦的是，这些问题通常不会在第一步暴露。Agent 可能先跑通一个小任务，然后在修改某个文件时突然踩到项目特有的约束，最后只能回滚重来。每次都靠人工在会话开头重新讲一遍项目结构，成本高且不稳定。

OpenClaw 的 Agent 即使挂载了 MCP 工具，也缺少“工作区本地知识”。全局配置解决不了每个项目的差异。AGENTS.md 就是用来补这一层的：它是一份放在工作区根目录的静态说明书，让 Agent 在动手前先读，把隐性约定显性化。

## 问题：不是工具不够，而是工作区没有边界感

很多 OpenClaw 用户会先把精力放在调 system prompt、加工具权限、接更多 MCP server 上。但实际卡住任务的，往往不是能力问题，而是 Agent 不知道这个工作区的边界在哪里。

比如：

- 项目要求用 `pnpm --frozen-lockfile`，Agent 却跑了 `npm install`；
- 本地有 `scripts/dev.sh` 一键启动，Agent 却自己拼了一条缺环境变量的命令；
- 某些目录是生成产物，不应手动修改，Agent 却直接编辑；
- 测试必须带 `--runInBand`，否则会端口冲突或超时。

这些细节一旦缺失，Agent 的每一步都像在陌生环境里试错。AGENTS.md 的价值，就是为 Agent 提供一份低噪声的“入职文档”。

## 做法：把工作区手册写成 Agent 会读的样子

### 1. 在工作区根目录创建 AGENTS.md

不要放在子目录，也不要散落在多个 README 里。Agent 查找成本越低，越可能真正执行。

### 2. 只写四块内容

一份能用的 AGENTS.md，通常不需要长。建议按以下结构写：

```markdown
# AGENTS.md

## Workspace map
- `src/`：主要源码，业务逻辑在这里
- `scripts/`：本地开发脚本，不要放测试文件
- `generated/`：自动生成，禁止手改

## Commands
- install：`pnpm install --frozen-lockfile`
- dev：`pnpm dev`
- test：`pnpm test -- --runInBand`
- build：`pnpm build`

## Constraints
- Node 版本固定为 20.x，不要升级
- 禁止直接修改 `generated/` 下任何文件
- 端口 4318 已被本地服务占用，测试不要使用
- `.env.example` 是模板，不要写入真实密钥

## Pre-task checklist
- [ ] 确认当前分支是否匹配任务
- [ ] 检查 AGENTS.md 是否有更新
- [ ] 运行测试前先确认本地服务是否正在运行
```

### 3. 让 OpenClaw Agent 先读后做

如果 OpenClaw 支持 workspace rules，就把 AGENTS.md 指过去；如果不支持，可以在任务模板或 system prompt 中明确：

> 进入工作区后，先读取 AGENTS.md，再使用任何写操作工具。

也可以在自动化任务模板里显式引用：

```text
先阅读 @AGENTS.md，按其中的 Workspace map 和 Constraints 执行。
```

### 4. 纳入版本控制

AGENTS.md 不是个人笔记，应该和代码一起提交、一起 review。当项目结构或命令变化时，同步更新它。

## 踩坑点

### 1. 文件太长，Agent 不读或读歪

AGENTS.md 超过 150–250 行后，被忽略或摘要失真的概率明显上升。不要把它写成项目百科。超出的内容可以拆成链接，比如 `docs/agent/setup.md`。

### 2. 写绝对路径

`/home/user/xxx` 这种路径换台机器就失效。尽量用工作区相对路径，或者用环境变量替代。

### 3. 和全局规则冲突

全局 OpenClaw 配置里允许某个 MCP 写操作，但项目里禁止。Agent 会犹豫，甚至选择服从全局默认。需要在 AGENTS.md 里标明优先级，例如：

> 本文件中的约束覆盖 OpenClaw 全局默认；未提及的部分遵循全局规则。

### 4. 把敏感信息写进去

Token、内网密码、私有仓库地址都不要放进 AGENTS.md。Agent 可能把文件内容带上日志或外部请求，泄露风险很高。

### 5. 只写不维护

过时的 AGENTS.md 比没有更糟。Agent 按旧命令执行会直接失败，用户还要花时间排查是不是 agent 变笨了。每次改启动命令、环境变量或目录结构时，顺手更新。

## 可复用建议

- **分层管理**：全局 OpenClaw 规则负责通用行为，AGENTS.md 只写工作区差异。不要让一份文件承载所有规则。
- **用脚本校验关键项**：可以加一个 `npm run check:agents`，检查 AGENTS.md 中提到的重要路径是否存在、关键命令是否可执行。
- **把约束写在首尾**：模型对文件开头和结尾更敏感。把“禁止操作”或“最高优先级约束”放在文件开头，把检查清单放结尾。
- **MCP 工具写清楚**：如果项目挂载了 MCP server，写明哪些工具允许直接调用，哪些需要用户确认，避免误触发写操作。
- **给 Agent 设开始前三问**：我在哪个工作区？AGENTS.md 在哪里？哪条规则优先级最高？这比堆更多 prompt 更有效。

## 总结

AGENTS.md 不是为了给 Agent 增加一套新规范，而是把工作区里那些“只有人知道”的隐性知识，变成 Agent 能稳定读取的边界信息。它的目标不是让 Agent 变得更聪明，而是减少它在陌生环境中的随机试探。

对 OpenClaw 用户来说，一份准确、简短、有明确边界的 AGENTS.md，往往比反复调整 prompt 或增加 MCP 工具更稳定。先把工作区手册写好，再让 Agent 动手，会少很多来回返工。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8b74800aef585f33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/13e829db541ee9b5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/355c8b7a80fff7cf.png)

