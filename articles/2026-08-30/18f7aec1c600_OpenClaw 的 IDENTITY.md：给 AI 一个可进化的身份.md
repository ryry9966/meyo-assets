---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35372
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw + Agent + MCP + 插件的组合里，很多任务不是一轮问答，而是长时间操作：读文件、调用工具、跑批处理、维护工作区。常见痛点是，每次新会话都要重新说明身份、边界、工具偏好；任务跑到一半，Agent 开始自由发挥，风格和约束逐渐漂移。

IDENTITY.md 是 OpenClaw 里用来沉淀 Agent 稳定身份和近期演进记录的文件。它不替代系统提示词，而是把跨会话需要保留的东西抽出来，让 Agent 在初始化时读取，必要时按规则更新。

## 问题

1. **会话失忆**：新会话里 Agent 不知道之前的角色、默认工具、语气和禁区。
2. **提示词膨胀**：把所有规则都塞进 system prompt，越写越长，维护成本高，冲突也不容易发现。
3. **偏好漂移**：长时间任务中，Agent 会逐渐偏离最初约定，比如开始过度解释、改变日志格式、调用不应该调用的插件。
4. **多工具不一致**：MCP 工具、插件和自动化脚本各自需要一份上下文，但缺少一个共同的身份源。

## 做法 / 步骤

### 1. 先放一个最小模板

不要一开始就写得很重。建议只保留几类信息：

```markdown
# IDENTITY

- role: OpenClaw 自动化操作员
- scope: 当前工作区文件读取、MCP 工具调用、批处理
- hard_constraints: 不读取密钥文件，不执行破坏性命令，除非明确批准
- style: 简体中文，简洁，先结论后步骤
- current_focus: 空
- evolution_log: []
```

### 2. 让 OpenClaw 初始化时加载

把 `IDENTITY.md` 放在工作区根部，或者配置 OpenClaw 指向固定路径。初始化时让 Agent 先读一遍，而不是把所有内容复制进 system prompt。系统提示词里只放一句“启动后先读取 IDENTITY.md，并遵守其中的 hard_constraints”。

### 3. 拆开读写权限

不要让 Agent 随意重写整个文件。建议暴露两个入口：

- `read_identity`：只读，初始化时调用。
- `append_identity_log`：追加式写演进日志，例如记录偏好变化、踩坑结论、工具选择原因。

稳定字段的修改需要人工 review，或者在 git diff 中确认。

### 4. 用版本和分隔线控制更新

在文件底部增加 `## Evolution Log`，每次只追加，不覆盖：

```markdown
## Evolution Log

- 2025-01-11: 发现 MCP 文件查询工具在大量小文件场景下变慢，批量任务优先用本地 grep。
- 2025-01-13: 用户偏好输出时先给结论，再给步骤。
```

这样 Agent 有了“可进化的身份”，但进化过程可追溯。

### 5. 与插件/MCP 工具对齐

如果插件需要读取偏好，可以让它读 `IDENTITY.md` 中的某个稳定段落，例如 `tool_preferences`。不要在每个插件里复制一份规则，否则很快会不同步。

## 踩坑点

- **身份膨胀**：大量旧偏好和过时结论会让 Agent 越来越保守，甚至套用不相关经验。建议每次压缩 `Evolution Log`，只保留最近 20 条或最近 30 天。
- **自我强化**：Agent 会不断给自己写偏好，最后形成“风格茧房”。关键字段加 `last_reviewed` 或 `expires_at`，到期后要求重新确认。
- **并发写入**：多个 Agent 或多个会话同时写 `IDENTITY.md` 会丢失更新。只允许一个 writer，或者使用 `version` 字段配合锁。
- **密钥泄露**：不要把 API key、token、数据库密码写进身份文件。身份文件可能被分享、提交到仓库或喂给其他工具。敏感信息必须保留在 secret 管理或环境变量中。
- **覆盖系统约束**：IDENTITY.md 不能凌驾于 OpenClaw 的基础安全策略之上。硬约束应同时存在于系统配置里，IDENTITY.md 只是补充和记忆，不是唯一安全边界。

## 可复用建议

- 把 `IDENTITY.md` 当代码维护：纳入 git，每次修改看 diff。
- 保持短小：稳定身份 8–15 行，演进日志尽量控制在屏内。
- 用“原则”而不是大量示例：示例一多，Agent 容易死记而不再理解。
- 每次会话结束时让 Agent 只追加一条“本次最值得记住的经验”，而不是无差别记录。
- 任务完成后清空 `current_focus`，避免下个任务被上一个任务的残留状态干扰。

## 总结

IDENTITY.md 的价值不是让 AI 更像人，而是给 Agent 一个可审计、可回滚、可压缩的持久身份层。对 OpenClaw/Agent/MCP/插件用户来说，它更像一个“跨会话的工程缓存”：小文件、明确边界、可写但受控。不要把它写成第二份 system prompt，而要让 Agent 能够安全地记住该记住的事，并随时知道哪些约束不可改变。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6ad23446b374e515.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d6207ddd9ddd4334.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f4ad48ea945f489c.png)

