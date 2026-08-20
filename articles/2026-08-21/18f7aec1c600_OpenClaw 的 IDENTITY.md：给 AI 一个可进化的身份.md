---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 33964
source: 综合讨论
publishedAt: 2026-08-21
---

在 OpenClaw 里跑 Agent，大家习惯把角色说明写在系统提示或启动参数里。短期够用，但任务一多，Agent 的行为就会漂：同一套 prompt 既要管风格、又要管边界，还不能随项目经验更新。接上 MCP 工具、插件多了以后，上下文里的“我是谁、该做什么、不该做什么”会越来越模糊。

## 问题在哪

系统提示通常是静态的。每次会话 Agent 从零开始读一遍，但它不会把“上次在这里踩过坑”“这个项目倾向用 pnpm 而不是 npm”沉淀下来。即使手动改了 prompt，下一次任务又可能覆盖。缺少一个轻量、可版本管理、允许 Agent 自己提议更新的身份文件。

IDENTITY.md 就是干这个的。

## 做法：把身份文件放进工作区

### 1. 建立最小模板

在项目根目录放一个 `IDENTITY.md`，一开始不要写太长。建议只保留几个字段：

- **Role**：Agent 在本项目里的角色
- **Scope**：允许做、不允许做的事
- **Defaults**：工具链、命令、路径约定
- **Known Pitfalls**：当前项目已知坑
- **Last Updated**：最后更新时间

示例：

```markdown
# IDENTITY

## Role
你是本仓库的 OpenClaw 自动化维护者。

## Scope
- 可以：运行测试、修改文档、提交脚本
- 不可以：直接推送到 main、修改 CI 密钥

## Defaults
- 包管理：pnpm
- 测试命令：pnpm test
- 禁止全局安装

## Known Pitfalls
- 测试依赖本地 Redis，需先启动 docker-compose
- Node 版本固定 20

## Last Updated
2025-01-01
```

### 2. 让 OpenClaw 在启动时读取

在 OpenClaw 配置或启动脚本里，把 `IDENTITY.md` 加入初始上下文。简单做法是：

```bash
IDENTITY=$(cat "$WORKSPACE/IDENTITY.md")
```

然后拼进 system prompt。更工程化一点，可以用 OpenClaw 的插件机制或 MCP 的 resources 暴露这个文件，让 Agent 每次新会话自动获取。

### 3. 更新走“提议 - review - 落盘”

直接让 Agent 改身份文件很危险。推荐流程：

- Agent 发现需要更新身份时，不直接覆盖 `IDENTITY.md`，而是写一个 `IDENTITY.proposed.md` 或输出 diff 建议。
- 人工 review 后合并到 `IDENTITY.md`。
- 如果必须自动更新，只允许追加 Known Pitfalls，不允许随意改 Role/Scope。

这样身份可进化，但不会失控。

### 4. 纳入版本管理

`IDENTITY.md` 和 `IDENTITY.proposed.md` 都放入 git。每次更新走 PR，保留历史。这样能追踪 Agent 的“认知”变化。

## 踩坑点

- **身份文件和任务数据混在一起**：有人把待办、临时上下文也写进 IDENTITY.md，文件越来越长，Agent 读取时注意力被稀释。身份文件只放稳定约束，不放具体任务状态。
- **Agent 自己改 Role**：某个任务失败后，Agent 可能“建议”扩大权限来绕过限制。如果自动合并，身份就被污染。Role/Scope 必须人工 review，甚至锁定。
- **频繁写入导致 git 噪音**：每次会话都改 Last Updated 没意义。只在内容真正变化时更新。
- **并发写冲突**：多个 Agent 实例同时跑，都提议更新同一个文件。建议用 `IDENTITY.proposed.<timestamp>.md` 或统一走队列，避免互相覆盖。
- **隐私和敏感信息**：身份文件容易写上 token、内网地址、用户名。不要放秘密，放入 .env 或专门的 secrets 管理。

## 可复用建议

- **字段最小化**：Role、Scope、Defaults、Known Pitfalls 就够了，别加“心情”“当前任务”等。
- **更新规则写清楚**：在 IDENTITY.md 里加一段 `## Update Policy`，说明什么能自动改、什么必须人工改。
- **用 MCP 资源暴露**：如果 OpenClaw 支持 MCP，可以把 IDENTITY.md 作为一个 resource，配合 `resources/templates` 让 Agent 读取和提交更新，比 shell 拼 prompt 清晰。
- **定期压缩**：每两周或每个里程碑，人工清理 Known Pitfalls，把不再适用的条目删掉，避免过时认知残留。
- **和 memory 分开**：IDENTITY.md 是身份约束，不是长期记忆。事件日志、任务结果放单独的 memory/日志文件，保持身份层干净。

## 总结

IDENTITY.md 的价值不在于“让 AI 更像人”，而在于给 Agent 一个可审计、可回滚、可进化的身份入口。对 OpenClaw 用户来说，它比无限加 prompt 更可持续。做得足够小、更新足够克制，才能真正用起来。

可以在仓库里加一个 `scripts/identity-check.sh`，在 pre-commit 时校验身份文件字段完整。这样更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/a36d0558b6ac34cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/2ade3a3a0b4cdea0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/4d2c0b06d278cc14.png)

