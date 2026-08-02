---
title: 让 AI 的身份证自己长大：OpenClaw IDENTITY.md 实战笔记
feedId: 31326
source: 综合讨论
publishedAt: 2026-08-02
---

# 让 AI 的身份证自己长大：OpenClaw IDENTITY.md 实战笔记

## 背景：Agent 也需要一本可修订的护照

用 OpenClaw 跑过多步任务的人都会遇到同一个尴尬：前 30 分钟 Agent 还兢兢业业执行你的 Conventions，3 小时后它开始用另一种风格写代码，甚至完全忘了最初的目标。这不是模型能力问题，而是**身份信息在长上下文里被稀释了**。

OpenClaw 提供了一种工程化解法：通过仓库中的 `IDENTITY.md` 文件，为 AI 赋予一个**结构化、可自我更新、带版本管理**的身份定义。它不是系统性 prompt 的简单外挂，而是一份可以被 Agent 自己读取、修改、提交的“活文档”。本质上，它把 Agent 的自我认知从一次性注入变成了持续演进的闭环。

## 问题拆解：静态提示词的三个致命缺陷

1. **目标漂移**：长任务中，原始指令被后续工具输出、思考链冲刷后，优先级隐性下降。
2. **知识冻结**：身份信息写死之后，Agent 无法把执行过程中获得的经验沉淀为行动原则，下次照样踩坑。
3. **多人协作割裂**：不同的开发者对同一个 Agent 的行为期望不一致，但提示词缺乏共同锚点。

OpenClaw 利用 Git 作为记忆后端，把 `IDENTITY.md` 当作“可 commit 的自我意识”，正好逐一解决这些问题。

## 实践步骤：让 IDENTITY.md 活起来

### 1. 初始化一个最小可行身份

在 OpenClaw 项目仓库根目录创建 `IDENTITY.md`，不要一上来就填成长篇小说。建议从这三个锚点开始：

```markdown
# Identity
- role: backend developer for OpenClaw plugins
- tone: pragmatic, minimal documentation
- constraints: no external dependencies unless approved
```

这三个字段能让 Agent 在行为边界内稳定发挥。角色定义能力域，语气约束输出风格，约束条件则划定工程安全边界。

### 2. 绑定 OpenClaw 的自更新规则

在 `openclaw.yaml`（或对应的配置）中指定记忆策略：

```yaml
identity:
  file: IDENTITY.md
  auto_update: true
  update_prompt: |
    Based on the last task execution, suggest a concise update to IDENTITY.md
    to reflect learned preferences or pitfalls. Output only the updated file content.
```

关键在于 `update_prompt` —— 它决定了 Agent 会把什么样的经验写回身份。我习惯强制要求输出**完整文件内容**而非 diff，避免后续合并时出现格式断裂。

### 3. 用 Git 管理身份变更

OpenClaw 会在任务结束后自动 commit 更新，但你最好主动配置一个专用分支策略：

```yaml
identity:
  branch: identity/auto-update
  create_pr: true
```

这意味着每次身份进化都会生成一个 PR。团队成员可以在合并前审查“AI 对自己的修改”，就像 code review 一样。曾经遇到过一次：Agent 在执行一连串报错后，把 `constraints` 里的“no external dependencies”自己改成了“prefer widely-used libraries”。这个改动在 PR 里引发了讨论，最终我们修改为更精确的表述后合并——这本身就是一种人机协同的认知对齐。

## 踩坑记录：三个容易翻车的地方

### 坑1：身份膨胀导致指令污染
Agent 倾向于持续追加内容，10 次任务后 `IDENTITY.md` 可能从 20 行暴涨到 200 行，包含大量冗余注意事项，反而降低注意力密度。  
**解法**：在 `update_prompt` 中加入长度限制，例如“Keep the file under 50 lines, prioritize removing outdated preferences over adding new ones”。必要时用定时任务压缩历史版本。

### 坑2：循环自指造成的认知锁定
如果更新规则过于宽松，Agent 可能把某个临时的应急行为固化为永久身份。例如一次任务中用了硬编码路径，就被写进“always use absolute paths”，后续任务完全僵化。  
**解法**：在身份模板中区分 `core`（不可自动修改）和 `learned`（允许自动更新）两个区块。核心身份由人维护，习得经验由 AI 填充，且标记时间戳，超过 7 天未再次命中的条目自动过期。

### 坑3：格式破坏导致解析失败
Agent 偶尔会输出“改进版” Markdown 破坏了原有结构，比如删掉了 YAML front matter 或改了字段名，导致 OpenClaw 后续读取身份失败，Agent 退化为无身份状态。  
**解法**：在 CI 中加入校验脚本，检查 `IDENTITY.md` 是否包含必需的字段（role, tone, constraints），并在格式错误时自动回复上一版。我们在 GitHub Actions 里配置了 `json-schema` 验证，效果很好。

## 可复用建议

- **结构化优于长篇大论**：用 YAML 或严格 Markdown 表格定义身份字段，便于解析和校验。
- **区分 immutable 和 mutable 区域**：用注释标记 `<!-- CORE_START -->` 和 `<!-- LEARNED_START -->`，让 Agent 和人都有清晰边界。
- **绑定项目 Conventions**：如果项目有 `CONVENTIONS.md`，在 `IDENTITY.md` 中用引用而非复制，避免双写不一致。
- **建立身份回滚机制**：保存最近 3 个版本快照，当 Agent 行为异常时可通过命令 `@openclaw identity rollback` 快速恢复。

## 总结

`IDENTITY.md` 不是另一个需要维护的文档负担，而是一个**认知加速器**。它让 Agent 不再每次都从零开始理解“我是谁、我要怎么做事”，同时也让团队能够显式地观察和校正 AI 的行为准则。

在工程实践中，最值得投入精力的不是写出一份完美的初始身份，而是设计**一个好的自更新规则和校验流水线**。当你的 Agent 在凌晨自动提交了一个 PR，只为了在 `IDENTITY.md` 里加一句 “Avoid using pandas for simple CSV parsing, python built-in csv is enough”，你就知道这玩意儿真的在干活了——而且带着项目该有的技术品味。

下一步可以尝试将 `IDENTITY.md` 与 MCP 工具链打通，让 Agent 把外部环境反馈也结构化地沉淀进身份，但这又是另一个值得专门写一篇的话题了。

---

