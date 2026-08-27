---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 34958
source: 综合讨论
publishedAt: 2026-08-28
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

在 OpenClaw 里接 MCP、插件和自动化任务时，常会遇到一个现象：AI 的角色边界会漂移。今天的会话里它只做文件整理，明天却可能因为某个工具可用，顺手执行了预期之外的写操作或网络调用。每次会话都要重新说明“你是谁、不能做什么、优先用哪个工具”，既浪费 token，也无法沉淀。

原因通常不是模型能力不足，而是身份信息被散落在 system prompt、会话开头、项目 README 或临时指令里。IDENTITY.md 的逻辑，就是把这些相对稳定的信息放进一个可版本化、可加载、可 review 的单一文件。它不负责技能实现，只负责定义“这个 agent 是谁、边界在哪、遇到冲突时怎么决策”。

## 问题

如果不用 IDENTITY.md，至少会碰到三类麻烦：

1. **角色漂移**：同一套 OpenClaw 配置，在不同会话中表现不一致。
2. **上下文污染**：每次手动注入大段身份说明，挤占工具返回和任务上下文。
3. **不可回滚**：身份规则改坏了很难发现，也不容易回到上一个稳定版本。

这些问题在自动化任务越多、MCP 工具越丰富时越明显。

## 做法 / 步骤

### 1. 定位身份文件

OpenClaw 常见布局是用户级 `~/.openclaw/identity.md` 与项目级 `.openclaw/identity.md` 结合。用户级放长期稳定身份，项目级放当前项目约束。若你的配置使用 `openclaw.yaml`，确认其中指向的 identity 文件路径，避免写错位置。

### 2. 按模板写 IDENTITY.md

建议内容分为四块：

- **身份核心**：名称、定位、职责范围。
- **硬边界**：禁止做什么，例如不调用写操作、不删除文件、不接触网络上传。
- **工具偏好**：同类任务优先走哪个 MCP 工具、哪个插件；哪些动作需要人工确认。
- **失败处理**：遇到权限不足、工具失败或信息不足时，是停止询问还是降级处理。

示例结构如下：

```markdown
# Identity
Role: local automation agent
Scope: file organization, note processing, scheduled tasks

## Hard limits
- Never delete files without explicit confirmation per run
- Never send local content to remote APIs

## Tool preferences
- Prefer filesystem MCP for reads
- Use notes tool for markdown creation

## Failure policy
- If uncertain, stop and ask
```

### 3. 让 OpenClaw 加载并验证

保存后重启会话或触发 reload。用一组固定探测问题验证是否生效，例如：

- “你是谁？现在负责什么？”
- “如果让你删除某目录，你会怎么处理？”
- “读取文件时优先用哪个工具？”

如果回答符合文件内容，说明加载成功；如果仍然泛泛而谈，检查文件路径或配置是否启用。

### 4. 按需迭代

改动 identity 时，不要直接让 AI 重写整份文件。只修改具体段落，并用 git 管理。每次变更记录原因，方便回滚。

## 踩坑点

- **文件太长**：把大量操作手册塞进 IDENTITY.md，会导致注意力稀释。身份文件应保持在一屏到两屏内，细节放独立 skill 或工具说明。
- **多级身份冲突**：用户级和项目级同时存在时，要明确优先级。项目级一般覆盖用户级，但硬边界不能被项目级放宽。
- **自动生成导致膨胀**：让 agent 自己总结身份时，容易写出“细心、强大、善解人意”这类无效描述。最好用行为化句式，例如“遇到只读请求不得写文件”。
- **修改后未 reload**：很多缓存场景下，文件改了但当前会话还沿用旧身份，容易误以为规则无效。
- **权限边界过宽**：只写“谨慎操作”不如写“删除前必须逐条确认”。抽象词对 agent 的约束很弱。

## 可复用建议

1. **身份分层，不要 all-in-one**：核心身份、项目身份、会话临时说明分开。
2. **用行为描述，不用形容词**：能测试的行为比抽象人设更可靠。
3. **把 identity 纳入版本控制**：每次行为异常时先看身份 diff。
4. **和 MCP 工具策略绑定**：在身份文件中直接写“只读任务用 filesystem MCP，写操作需确认”。
5. **定期身份回归**：准备固定问句，每两周或每次大改后跑一遍，确认边界未失效。

## 总结

IDENTITY.md 不是又一个 prompt 文件，而是 OpenClaw agent 的“运行时宪法”。它可进化的前提是：文件足够短、行为足够具体、变更可回滚、加载可验证。真正做到这些后，你会发现 agent 的稳定性和可维护性提升，不是因为模型更强，而是因为边界终于被管住了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8101b25dbf4f933b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/24b9fa7dc5734aaa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/224eec94ecfc0c4b.png)

