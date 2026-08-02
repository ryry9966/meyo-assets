---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 31423
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景：为什么 System Prompt 不够用了

在基于 OpenClaw 构建 Agent 集群或 MCP 工具链时，我们一开始总是把角色定义、行为约束、领域知识塞进 `system_prompt` 字段里。这在单 Agent、短会话的场景下勉强能工作，但很快会暴露出三个工程层面的问题：

1. **上下文失忆**：Agent 重启、会话切换后，之前纠正过的行为模式、新增的规则全部丢失，需要人工重新注入。
2. **膨胀的 Prompt**：随着业务迭代，system prompt 突破 2000 tokens，不仅推理质量下降，还挤压了宝贵的上下文窗口。
3. **多 Agent 协作冲突**：不同 Agent 共享部分人格，但又需要差异化的领域知识，静态 prompt 很难实现“复用公共身份 + 叠加私有配置”的模式。

OpenClaw 给出的解法之一，是将身份信息外挂到文件系统，并允许 Agent 在运行时有条件地读取、甚至更新这个文件。这个文件就是 `IDENTITY.md`。

## 不是另一个 Prompt 文件

`IDENTITY.md` 的核心设计思路是：**把“我是谁”从指令集里剥离出来，变成一个可持久化、可版本控制、可自更新的状态文件**。它与传统 system prompt 的差异在于：

- 它存放在 Agent 的工作目录或 OpenClaw 的配置注册表中，不被直接注入到 prompt。
- Agent 需要时通过内置工具（如 `identity_load`、`identity_diff`）去读取，而不是每次调用都携带全文。
- 在显式授权的情况下，Agent 可以写回 `IDENTITY.md`，实现“经验积累”。

这样一来，身份信息从“一次性传参”变成“持久化状态 + 按需加载”，长度不再受限，同时为多 Agent 身份继承提供了基础。

## 工程化的第一步：IDENTITY.md 的结构

从业界实践和 OpenClaw 的默认模板看，推荐的最小规范包含四个区块，直接使用 Markdown 二级标题分隔：

```markdown
# IDENTITY

## Meta
id: agent-oss-insight
version: 4
last_updated: 2025-03-17T09:12:00Z
sync_policy: manual

## Persona
你是一名面向开源社区的技术分析师。语气简洁、保守，不预测趋势，只做归因和复现描述。

## Knowledge
- 目标仓库：openclaw/core, openclaw/plugins
- 已知陷阱：plugin 初始化时 Runtime 环境变量 CL_BACKEND 为 linux 时必须使用 /tmp 作为暂存区，不要使用 /dev/shm

## Behavior Rules
- 回答前检查 Knowledge 中的已知陷阱是否适用于当前场景
- 如果被问及未观测到的数据，直接回复“无相关数据”，不编造
- 每次对话结束时，如有新发现，向用户提议更新 Knowledge 区块
```

`Meta` 块用于版本追踪和同步策略，`Persona` 和 `Knowledge` 分别维护性格与事实，`Behavior Rules` 则是可操作的行为约束。最重要的是留下一个“更新”的出口，否则 `IDENTITY.md` 又会退化成静态文件。

## 让 Agent 读写 IDENTITY.md：步骤与示例

假设你的 OpenClaw 项目结构如下：

```
project/
├── agents/
│   └── analyst/
│       ├── agent.yaml
│       └── IDENTITY.md
```

在 `agent.yaml` 中启用 identity 插件：

```yaml
identity:
  enabled: true
  path: ./IDENTITY.md
  allow_update: true
  update_confirm: always   # 必须用户确认后写入
```

在 Agent 会话中，调用流程大致是：

1. **加载身份**：Agent 首次处理任务时通过内置命令 `!identity-load` 将文件内容拉入工作记忆。
2. **发现新知识**：运行中识别到可复用的模式（比如某个 API 的超时阈值），向用户提议：“建议将新知识写入 Knowledge。”
3. **用户确认后更新**：用户回复 `!identity-update`，Agent 执行差异合并，更新 `Meta.version` 和 `last_updated`，并写回文件。

关键点是 **人机协同的更新回路**，不要直接开放自动无审核写入，否则一次幻觉就会污染整个身份文件。

## 踩坑点与规避

在实践中，我们踩过三个典型的坑：

### 1. 文件编码与解析不一致
特定版本的 YAML 解析器和 Markdown 渲染器对 UTF-8 BOM、行尾换行符敏感。在一个团队中，有人用 Windows 编辑 `IDENTITY.md` 带入了 CRLF，导致 Agent 的 `identity_load` 工具解析 `last_updated` 时间戳出错。**统一使用 LF 换行 + 无 BOM 的 UTF-8**，并在 CI 中加入 `dos2unix` 检查。

### 2. 循环更新导致身份漂移
在允许自动更新且未设置 `update_confirm` 的场景中，Agent 在一次长任务中反复追加“优化建议”，最终 Knowledge 区块中出现自相矛盾的规则（例如同一 API 的超时从 30s 到 5s 又回到 10s）。**必须保留用户确认环节**，并且考虑启用 `identity_diff` 让用户看到 delta，而非直接接受全文覆盖。

### 3. 多 Agent 共享目录的并发写入
如果将多个 Agent 指向同一个 `IDENTITY.md` 且不控制写入锁，会出现文件截断。解决办法不是加文件锁（OpenClaw 当前版本未提供），而是**为每个 Agent 建立独立身份文件，公共部分通过模板继承**。可以用脚本在部署前从 `common/identity_base.md` 生成各 Agent 的身份文件，并标记 `sync_policy: inherits`。

## 可复用的工程建议

1. **版本控制是生命线**  
   把 `IDENTITY.md` 纳入 Git，每次更新前自动 commit。这样即使 Agent 写入了错误信息，也可以通过 `git diff` 快速回退。

2. **分层设计身份文件**  
   将公共的 Persona、Behavior Rules 抽离为 `identity_base.md`，各 Agent 的 Knowledge 独立维护，避免信息耦合。在构建时用脚本拼装，生成最终的 `IDENTITY.md`。

3. **监控身份文件的“健康度”**  
   编写简单的 Python 脚本（可集成进 CI）检查 `IDENTITY.md` 的大小是否超过阈值（如 10KB）、版本号是否递增、必要区块是否存在，防止静默损坏。

4. **与 MCP 工具链集成**  
   如果你在 MCP 工具中实现了自记忆功能，可以让工具返回结构化的“经验对象”，然后 Agent 将其归并到 Knowledge 中。关键是保持 Knowledge 为简洁的事实列表，而非大段叙述，以便工具检索。

## 从静态 Prompt 到可进化身份

OpenClaw 的 `IDENTITY.md` 提供了一种轻量但有效的工程手段，让 Agent 在长期运行和多人协作中保持角色一致性，并具备有限的自学习能力。它不追求复杂的向量记忆库，也不依赖外部数据库，仅用一个 Markdown 文件 + 确认机制，就能显著减少重复的 prompt 调校工作。

如果你正被 Agent “说变就变”的毛病困扰，不妨从一个结构清晰的 `IDENTITY.md` 开始，把知识沉淀下来，让 AI 从“一次性工具人”变成“你团队里有记性的协作者”。

---

