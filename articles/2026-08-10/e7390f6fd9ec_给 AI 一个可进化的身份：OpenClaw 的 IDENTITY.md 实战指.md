---
title: 给 AI 一个可进化的身份：OpenClaw 的 IDENTITY.md 实战指南
feedId: 32296
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么你的 Agent 总是“失忆”？

在 OpenClaw 多智能体编排中，最常见的抱怨不是模型能力不足，而是**行为一致性太差**。同样是代码审查任务，周一它能精准揪出竞态条件，周三却开始建议把 `for` 循环全改成 `map`，风格判若两人。根源不是 prompt 没写好，而是**缺少一个持久的、可进化的身份锚点**。

大多数实践者把身份定义硬编码在每次调用的 system prompt 里，或者分散在各处的 `manifest.yaml` 注释中。这带来三个工程债：

1. **无法积累经验** —— 每次新任务都在“从零驯化”一个 Agent。
2. **协作互相污染** —— 多人编写的 instruction 会出现风格冲突，Agent 行为如精神分裂。
3. **调试地狱** —— 当 Agent 行为异常时，你无法回溯它是“基于什么经验”做出的判断。

OpenClaw 的内置机制 `IDENTITY.md` 就是为解决这些问题设计的。它不是简单的 Markdown 公告板，而是一份**活的、允许 Agent 自我更新**的身份契约。

## IDENTITY.md 的工作原理

`IDENTITY.md` 位于项目（或 Agent 实例）根目录，由 OpenClaw 框架在每次任务开始时**自动注入到顶层 system prompt 的最前端**，优先级高于用户指令。其内容定义了 Agent 的：

- 核心人格：角色、语气、知识边界
- 行为准则：禁止做什么、必须遵守什么协议
- 长期记忆：已掌握的项目习惯、用户偏好、踩过的坑
- 成长日志：经过审查的自我修改历史

最关键的是，OpenClaw 允许在任务结束后，通过定义好的“身份更新策略”（通常在 `claw-project.yml` 中启用），让 Agent 评估本次任务中是否有值得沉淀的经验，并**向 IDENTITY.md 提交一个结构化的 PR 修改**，这就是“可进化”的由来。

## 实战步骤：从零构建一个可进化的 Agent 身份

### 1. 创建冻结层与进化层

不要把所有内容塞进一个文件。推荐将 IDENTITY.md 拆成两个逻辑区：

```
<!-- FREEZE:START -->
<!-- 此区块禁止 Agent 修改，由人类维护 -->
# 核心身份
你是一个专注于后端代码审查的 Agent，使用 Go 1.22 标准，语气简洁，禁用 emoji。
<!-- FREEZE:END -->

<!-- EVOLVE:START -->
<!-- 此区块允许 Agent 在审查后追加经验 -->
## 积累经验
- 2025-01-15: 用户明确禁止推荐 `interface{}`，优先使用泛型。
- 2025-01-16: 发现 Gin 框架的 `c.ShouldBindJSON` 在错误处理上需要额外注意状态码。
<!-- EVOLVE:END -->
```

OpenClaw 的更新工具（通过 MCP 服务器 `claw-evolution` 或内置的 `identity_updater` skill）会识别 `FREEZE/EVOLVE` 标记，确保核心人格不会漂移。

### 2. 启用身份进化流水线

在 `claw-project.yml` 中配置：

```yaml
evolution:
  identity:
    enabled: true
    strategy: review-then-apply
    max_entries_per_task: 2
    require_approval: true  # 生产环境务必开启
```

策略 `review-then-apply` 表示 Agent 在任务结束后生成一个修改建议，需要人类批准才会写入文件。如果你完全信任 Agent 的自主判断（仅推荐在内部工具中），可设为 `auto-apply`。

### 3. 给出好的“种子”

初始 IDENTITY.md 必须包含**足够具体的边界**。错误的写法是“你是一个好助手”，正确的写法是：

- 角色：“你是一个 Go 语言代码审查员。”
- 知识边界：“仅对 `./services/` 目录下的代码提供意见，禁止评论测试用例的命名风格。”
- 反例：“如果遇到不确定的并发模式，必须回复‘需要人工确认’，禁止猜测。”
- 进化指令：“当用户纠正你的建议超过两次次时，总结为一条经验记录到 EVOLVE 区域。”

种子越具体，进化路径越可控。

### 4. 用 MCP 工具手动触发进化

即使没有立即配置自动化，也可以通过 OpenClaw 的 MCP 接口手动让 Agent 回顾并更新身份。内置的 `claw-evolution` 提供 `summarize_session` 工具，将当前对话中的关键纠偏凝练为经验条目，你审核后通过 `upsert_identity_entry` 写入 EVOLVE 区。这对调试新 Agent 特别有用——运行几次对话，积累 5-10 条经验后，再试该开启自动进化。

## 踩坑清单：我经历过的翻车现场

**身份漂移最致命**  
一次我关闭了 `require_approval`，Agent 把用户的一句玩笑“你能不能别这么啰嗦”误识别为重大偏好变更，在 EVOLVE 区写入“用户要求永远只用 bullet point 回复”，导致它之后把所有代码块的解释全删了。从此我坚持：**任何一种自动修改策略，都必须搭配人类审批，除非任务域是玩具项目**。

**多余经验污染**  
进化不加限制时，Agent 会像囤积癖一样把每次交互都写成经验。一个月后 IDENTITY.md 膨胀到 200+ 条，system prompt 过长导致模型注意力崩塌。解决办法是设置 `max_entries_per_task: 1`，并定期人工清理过时条目。经验也要有淘汰机制。

**与系统指令的冲突**  
OpenClaw 的 system prompt 中还有来自 Manifest 的指令。如果 IDENTITY.md 中说“始终使用中文回复”，而 Manifest 中声明国际化，最终行为是 IDENTITY 胜出，因为它注入顺序更前。许多时候你会忘记修改 IDENTITY，排查起来费时。养成习惯：每次修改 Manifest 中行为相关的元信息，都用脚本检查 IDENTITY 中的冲突语句。

**文件格式被破坏**  
Agent 直接编辑 Markdown 时偶尔破坏 `FREEZE` 标记，或错误合并重复条目。务必在版本控制中配置 `pre-commit` 钩子，用正则校验 IDENTITY.md 结构完整性（例如 `FREEZE:START` 和 `FREEZE:END` 成对出现）。

## 可复用实践建议

1. **模板化**  
   维护一个 `identities/` 目录，放入 `code-reviewer.md`、`docs-writer.md` 等模板，新项目从模板拷贝并调整 `FREEZE` 区。

2. **进化日志即知识库**  
   定期将多个 Agent 的 EVOLVE 区域汇总，形成团队级别的“Agent 经验知识库”，用于训练或微调时构造示例数据。OpenClaw 社区已有人用此方法显著降低新 Agent 冷启动成本。

3. **串联 CI**  
   在评审流水线中，每次合并请求后触发一个轻量级任务，让代码审查 Agent 根据本次评审中被采纳/拒绝的评论，自动更新自己的 IDENTITY 经验区，并创建一个提交到仓库。这样 Agent 随着项目一起成长。

4. **多代理身份隔离**  
   在 OpenClaw 编排多个 Agent 时，每个 Agent 实例应有独立的 `identity/<agent_name>.md`，不要共用。因为进化依赖上下文隔离，共用只会互相干扰。

5. **回滚能力**  
   对 IDENTITY.md 执行版本控制，并养成打 tag 的习惯。比如 “identity-before-foo-refactor”，当 Agent 在新需求上行为退化时，能快速回滚到已知良好状态。

## 总结

IDENTITY.md 把 AI 从“一次性佣兵”升级成“项目的学徒”。它让 Agent 的行为不再仅仅依赖单次 prompt 技巧，而是建立在可追溯、可审查、可进化的长期记忆之上。整个过程不需要复杂的向量数据库或微调，纯靠文件约定和审批流就实现了工程化的 agent 成长机制。

对于已经在使用 OpenClaw 做自动化或插件编排的团队来说，投入一个小时设计好初始身份和进化策略，之后每个迭代的沟通成本会断崖式下降。同时建议始终以“先冻结、再进化、有人审”三原则为底线，这是我用三辆滑板车才换来的教训。

---

