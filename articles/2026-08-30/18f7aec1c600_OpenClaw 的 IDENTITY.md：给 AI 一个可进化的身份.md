---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35300
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 经常被当作个人助理、自动化执行器，同时接入 MCP、文件系统、脚本、浏览器等能力。工具越多，会话越容易漂移。与其把角色设定、语气、工具偏好、安全边界全部堆在 system prompt 里，不如放进一个专门的身份文件：`IDENTITY.md`。

这个文件不一定需要复杂机制。它更像一个运行基线：让 agent 在每次启动、切换 MCP server 或进入新任务前，先知道自己是谁、能做什么、不能做什么。

## 问题

1. **系统提示词膨胀**：角色说明、风格要求、权限边界、示例全混在一起，越改越难维护。
2. **跨会话行为漂移**：这轮像执行器，下轮像搜索引擎，输出风格不稳定。
3. **没有持久身份**：MCP 工具或插件上下文切换后，模型丢失原来的工作约定。
4. **更新靠手改**：Agent 不能从错误中沉淀规则，所有身份调整都依赖人工编辑。

## 做法/步骤

### 1. 建立最小身份文件

建议使用 YAML front matter 存结构化字段，正文只放不可变身份说明。保持文件在 200 行以内。

```yaml
---
name: openclaw-agent
version: 7
updated: 2025-01-01
role: personal automation assistant
tone: concise, engineering-oriented
boundaries:
  - never expose absolute paths
  - ask before destructive file operations
tools:
  preferred: [fs, mcp-fetch]
  avoid: []
memory:
  facts_dir: .openclaw/memory/facts.md
  prefs_dir: .openclaw/memory/prefs.md
---
```

正文只写身份说明，例如“你是 OpenClaw 的本地自动化助手，优先使用只读工具，遇到高风险操作先确认。”

### 2. 会话启动时注入

OpenClaw 启动后先读取 `IDENTITY.md`，再加载本次任务上下文。可以简单用脚本输出：

```bash
cat .openclaw/IDENTITY.md
```

如果通过 MCP 暴露，建议挂载为只读 resource。不要把身份文件放在任意可写目录，否则容易被误改。

### 3. 设计可进化的更新规则

不建议让模型直接覆盖身份文件。更好的方式是“提议—确认—合并”：

- 模型输出 YAML patch
- 只允许修改白名单字段：`role`、`tone`、`boundaries`、`tools`、`memory`
- 变更原因写入 `CHANGELOG.md`
- 版本号 +1

这样身份可以进化，但每次变更都可审计、可回滚。

### 4. 分离动态记忆与静态身份

身份不应该频繁变化，事实和偏好才需要更新。把动态记忆放到 `facts.md`、`prefs.md`，身份文件只引用路径。不要把聊天记录、临时任务写进 `IDENTITY.md`。

### 5. 加自检

每次身份更新后，跑几个固定问题：

- 你是谁？
- 遇到未知工具是否先确认？
- 是否会泄露本地路径？
- 输出语气是否稳定？

符合预期再保留版本。

## 踩坑点

- **身份文件过长**：超过 300 行后，模型容易忽略尾部。拆成 identity、runtime profile、memory。
- **允许直接覆盖**：模型可能把一次错误策略写进身份。必须经过 diff 或 allowlist 确认。
- **把 secrets 写进去**：token、密码写进身份后，文件变成高风险资产。只写环境变量名或引用路径。
- **与 system prompt 冲突**：如果 system prompt 说叫 A，IDENTITY 说叫 B，模型可能随机选择。加载优先级要明确：身份为基础，任务指令可覆盖执行细节，但身份字段不覆盖。
- **只增版本不测试**：更新后不验证，可能长期使用坏身份。需要保留上一个 stable 版本。
- **工具输出混入身份**：禁止将 MCP 工具返回内容直接追加进身份文件，除非经过结构化过滤。

## 可复用建议

- 把 `IDENTITY.md` 当“合同”而不是“剧本”：写边界，不写具体任务步骤。
- 每次只改一类内容：角色、工具偏好、记忆路径不要混在一起更新。
- front matter 便于程序解析，正文保持简短。
- 如果仓库会公开，先做脱敏，避免泄露个人工作流和目录结构。
- 配合 MCP 时，身份文件作为只读 resource 暴露，允许查询，不允许写入。
- 维护身份版本历史，方便对比“改了什么导致行为变化”。

## 总结

`IDENTITY.md` 的价值不是给 AI 一个漂亮人设，而是让它拥有可审计、可回滚、可进化的运行基线。先解决稳定，再谈进化；先限制写入，再允许更新。对 OpenClaw、Agent、MCP、插件和自动化实践者来说，这个文件应该作为基础设施的一部分维护，而不是 prompt 的草稿箱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/70ef5a42fd64c988.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bc2db17e90aa45a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ec7c7fd27d0f46ed.png)

