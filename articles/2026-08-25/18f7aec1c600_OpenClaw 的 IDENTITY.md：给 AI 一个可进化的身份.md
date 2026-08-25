---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 34626
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent 环境里，我们经常让同一个实例同时处理脚本、MCP 工具调用、插件维护和自动化任务。时间一长，问题会逐渐暴露：不同会话里的语气不一致，边界判断偶尔越权，已经踩过的坑在下次任务里继续犯。

这些问题通常不是因为模型能力不够，而是身份信息被分散在 system prompt、启动脚本、环境变量、README 和插件注释里。没有一个单一事实来源，每一次“人设调整”都靠临时指令或改代码来完成，无法版本化，也无法追溯。

## 问题

常见的痛点包括：

- Agent 的行为规则散落多处，发生冲突时不知道以哪份为准。
- 每次调整角色、边界或决策偏好，都要改启动脚本或 prompt 模板。
- 经验无法沉淀：上次明确验证过的失败模式，下次任务仍然会触发。
- 多会话、多任务场景下，Agent 的身份逐渐漂移，风格和口径越来越不稳定。

## 做法与步骤

我把身份治理收敛到一个仓库级文件：`IDENTITY.md`。它放在 OpenClaw 工作区根目录，作为 Agent 启动时强制读取的上下文。

### 1. 初始化并注入

在 OpenClaw 的 session starter 或 system prompt 模板中加入一条硬规则：

```text
Before any action, read IDENTITY.md. If there is a conflict between IDENTITY.md and other instructions, IDENTITY.md wins.
```

如果 OpenClaw 支持 MCP 资源读取，可以让 Agent 通过 resource 接口读取该文件，而不是把全文长期塞在 prompt 里，减少上下文膨胀。

### 2. 字段设计

我目前使用的 `IDENTITY.md` 模板大致如下：

```markdown
# Identity
- 角色定位
- 服务对象
- 默认语气

# Boundaries
- 允许执行的操作
- 禁止执行的操作
- 敏感操作需人工确认

# Decision Rules
- 工具选择优先级
- 冲突处理方式
- 失败重试策略

# Known Failure Modes
- 已验证的失败模式及避免方法

# Glossary
- 项目内专有名词、缩写

# Evolution Log
- 每次重大变更的原因和结果
```

这个模板不是标准答案，但核心原则是：**记录“为什么”和“边界”，不记录“怎么做”**。具体操作步骤放到技能文档或工具说明里。

### 3. 让身份可进化

身份文件不应一次写完就固定。可以设置固定节奏，让 Agent 在任务结束后输出一段 `identity_proposal`，只包含新增的失败模式或决策规则，不直接改主文件。

例如，可以在任务结束时要求：

```text
Based on this task, propose at most 2 additions to IDENTITY.md. Only include a rule if it was verified in this session and is likely to recur.
```

然后由人工 review，通过 git 合并到主分支。这样可以避免一次偶发错误被写成通用规则。

### 4. 版本控制

`IDENTITY.md` 纳入 git 管理，每次人工审核后 squash merge。文件内部只保留版本号和最近一次变更说明，详细历史交给 `git log`，避免文件自身变成冗长的 changelog。

## 踩坑点

### 1. 文件过长，注意力被稀释

Agent 对长上下文的中间部分关注度会下降。建议 `IDENTITY.md` 控制在 80–120 行以内，其余内容拆分到 `POLICY.md` 或 `SKILLS.md`。

### 2. 敏感信息被写入身份文件

不要把 API key、token、服务器地址写进 `IDENTITY.md`。它会被频繁读取，泄露风险高。敏感信息放 secret manager 或 `.env`，身份文件里只写“凭据读取规则”。

### 3. 规则过于刚性

如果所有边界都写成“必须/禁止”，Agent 遇到未覆盖场景时可能硬套规则，反而引发误操作。需要保留 fallback：

```text
If rules conflict or the situation is unclear, stop and ask the user before taking action.
```

### 4. 让 Agent 自动更新主文件

直接允许 Agent 修改 `IDENTITY.md` 很容易产生反馈噪音。一次网络超时可能被写成“所有 API 调用需要重试 5 次”。正确的做法是：Agent 写 proposal，人审核后合并。

### 5. 多 Agent 共用同一文件

不同角色的 Agent 共用一份 `IDENTITY.md` 时，容易出现边界冲突。建议按角色拆分，例如：

```text
GLOBAL.md   # 所有 Agent 共用
ROLE_OPS.md # 运维角色
ROLE_DEV.md # 开发角色
PROJECT.md  # 项目特定规则
```

## 可复用建议

- **分层治理**：全局规则、角色规则、项目规则分开，避免单文件膨胀。
- **记录边界而非做法**：`IDENTITY.md` 只写“能做什么、不能做什么、遇到冲突怎么办”。
- **用最小可验证条目**：只记录已验证且可复现的失败模式，避免把猜测写进去。
- **身份要带版本号**：在文件头部加入 `version` 和 `updated_at`，避免旧缓存造成不一致。
- **定期人工 review**：建议每周或每完成 N 个任务后，集中审核一次 `identity_proposal`，合并有效经验。

## 总结

`IDENTITY.md` 不是简单写一段人设文本，而是给 OpenClaw Agent 一个可控制、可回滚、可审计的身份事实源。它能把散落在各处的行为规则收敛起来，让 Agent 在长期任务中保持稳定，同时通过“建议—审核—合并”的循环，让身份随着项目一起演进。

真正的可进化，不是让 AI 随便改自己的身份，而是在版本控制和人工 review 的约束下，让经验持续沉淀。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/a4ed5b8ec673bd1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/076afc6631e663d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/feee726de4b361da.png)

