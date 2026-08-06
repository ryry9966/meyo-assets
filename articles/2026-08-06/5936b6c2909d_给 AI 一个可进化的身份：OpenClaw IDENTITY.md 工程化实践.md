---
title: 给 AI 一个可进化的身份：OpenClaw IDENTITY.md 工程化实践
feedId: 31838
source: 综合讨论
publishedAt: 2026-08-06
---

# 给 AI 一个可进化的身份：OpenClaw IDENTITY.md 工程化实践

## 背景

在 Agent 与自动化流程的工程化实践中，我们常用 OpenClaw 这类框架将 LLM 接入插件体系、MCP（Model Context Protocol）工具和外部知识库。智能体能执行复杂任务，但每次对话几乎都是从零开始：角色设定写在 System Prompt 里，一旦任务结束，AI 不会记得如何优化过某个流水线，也不知道你喜欢用 `pnpm` 还是 `npm`。于是我们不断重复相同的纠偏、偏好声明、错误排查。

OpenClaw 提出的 IDENTITY.md 机制，正是为了解决这类**“可进化身份”**的问题。这个文件不再是固定死的 prompt，而是一个由智能体自身维护的、随着交互持续更新的自我认知文档。它既包含静态角色定义，也承载长期记忆、决策偏好和技能积累，真正让 AI 从“一次性工具”变成“会成长的协作者”。

## 核心问题拆解

传统方案有三大痛点：
1. **身份碎片化**：角色描述分散在代码、环境变量、Prompt 模板中，难以统一维护。
2. **无记忆连续性**：Agent 无法跨会话保留个性化的执行经验。
3. **更新成本高**：想让 Agent 记住新偏好，必须手动修改配置或 Prompt，做不到自动学习。

IDENTITY.md 的设计目标正是将这三点打通：一个 Markdown 文件，结构清晰，人类和 AI 都能读写，由 OpenClaw 的记忆插件在每次任务后驱动更新。

## 实现步骤（以 OpenClaw v0.12+ 为例）

### 1. 在项目根目录创建 GUIDES/IDENTITY.md

```markdown
# AI 身份定义
name: dev-assistant
role: 资深后端 Node.js 开发者，专注 API 设计与自动化脚本
preferences:
  package_manager: pnpm
  test_framework: vitest
  commit_style: conventional
memories:
  - 2025-01-15: 修复了用户管理接口的 N+1 查询，改用 Dataloader
  - 2025-02-03: 用户偏好使用 Zod 进行运行时校验，而非 Joi
rules:
  - 所有数据库迁移必须附带回滚脚本
  - 生成的代码需要包含 JSDoc 类型注释
```

### 2. 在 OpenClaw 配置中挂载记忆插件

确保已安装 `@openclaw/plugin-memory`，并在 `agent.config.yaml` 中启用：

```yaml
plugins:
  - name: memory
    source: ./plugins/memory.ts
    identity_file: GUIDES/IDENTITY.md
    update_policy: on_session_end   # 会话结束后自动更新
    max_memories: 50
```

插件会读取当前身份，拼接进系统 prompt，并在对话流中捕获重要经验和新出现偏好，追加到 `memories` 和 `preferences` 中。

### 3. 利用动作注解让 Agent 主动更新身份

OpenClaw 的动作系统允许 Agent 显式调用 `update_identity` 函数：

```typescript
// 在 agent 定义里注册动作
actions:
  update_identity:
    handler: memory.updateIdentity
    description: 更新身份文件
```

然后 Agent 在对话中可以说：“我将记住你讨厌 GraphQL 嵌套过深的问题”，并自动调用该动作更新 IDENTITY.md。

### 4. 版本控制与人工审核

人类开发者可以定期检视 GUIDES/IDENTITY.md 的 diff，确认智能体写入的经验是否符合团队规范。建议将身份文件纳入 Git，每次更新作为一个 commit，并在 PR 中 review。

## 踩坑记录

**坑 1：记忆膨胀**  
早期我们放开 `max_memories` 限制，两周后文件超过 2000 行，Agent 开始“怀旧”，在无关任务中复述旧经验。解决办法：设置硬上限并用 LLM 定期做记忆摘要压缩，只保留高价值条目。

**坑 2：身份冲突**  
当多个 Agent 共享同一个 IDENTITY.md 时（如开发助手和测试助手），偏好互相覆盖。例如测试助手将 `test_framework` 改成 Jest，开发助手后续运行就出错。解法：用 namespaces 区分不同 Agent 的身份段，插件只读取/修改各自区块。

**坑 3：更新时机不当**  
`on_session_end` 更新若遇到会话异常终止，可能导致写一半的文件。我们改用原子写策略：先写 `.identity.md.tmp`，在插件中校验 Markdown 结构合法性后再 rename 覆盖，失败则回退。

**坑 4：隐私与安全**  
IDENTITY.md 会记录交互细节，比如 API 端点、数据库表名，如果仓库公开会造成信息泄露。务必让记忆插件支持敏感字段过滤或匿名化，并在 `.gitignore` 中排除实际生产的记忆文件，只保留模板。

## 可复用的工程化建议

1. **模板化身份结构**  
   为项目类型（API 服务、CLI 工具、文档站点）建立 IDENTITY.md 模板，新项目只需调整 `role` 和初始 `preferences`，降低冷启动成本。

2. **搭配知识库提升准确度**  
   别让 Agent 仅依赖记忆，将事实性知识（如公司编码规范、API 文档片段）放到向量知识库，IDENTITY.md 只存个性化经验。

3. **记忆质量评分**  
   简单加入一个 `confidence` 字段，Agent 写入记忆时可评估可信度。后续读取时按评分排序，优先提供高价值信息。

4. **定期清理与归档**  
   设置 cron job，将超过 30 天未使用且评分低的记忆移入 `archive/` 目录，防止身份文档退化。

5. **多模态记忆的扩展**  
   OpenClaw 插件支持将图片描述、代码 diff 概述等也存入记忆，可丰富 Agent 的上下文理解。

## 总结

IDENTITY.md 把 Agent 的“自我意识”从空中楼阁变成了一个可读写、可评审、可进化的工程文件。在 OpenClaw 的插件化架构下，实现成本很低，但对长期人机协作的效率影响巨大——你终于可以在第二天打开终端时说“继续昨天的重构”，而 Agent 能想起上次的进度和你的习惯。

实际落地时，把握好记忆的生命周期管理（压缩、过滤、命名空间隔离），这套机制就能从实验品变成可靠的工程基础。让 AI 拥有可进化的身份，本质上是在为你的自动化流程植入持续学习的基因。

---

