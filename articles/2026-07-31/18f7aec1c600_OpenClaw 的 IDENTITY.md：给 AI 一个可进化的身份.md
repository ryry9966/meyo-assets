---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 31088
source: 综合讨论
publishedAt: 2026-07-31
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景：当 Agent 失去“我是谁”

构建过长时间运行 Agent 的人都会撞上一堵墙——上下文窗口再长也装不下一个“持续自我”的模型。每次新对话都是从零开始，哪怕你昨天刚教会它只使用 `npm` 不用 `yarn`，今天它又会热情洋溢地建议 `yarn add`。短期记忆靠 system prompt 里硬编码，长期记忆靠“挂载一整个向量数据库”听上去很美，落地却经常卡在检索质量、冷启动延迟和维护成本上。

OpenClaw 作为一个允许 AI 执行本地命令、调用外部 MCP 工具、连接插件的运行环境，更需要一种**轻量、可审计、可手动修正**的长期身份方案。IDENTITY.md 就是为此设计的：一个放在项目根目录的 Markdown 文件，既是 AI 的“简历”，也是它的“工作日志”，并且可以被 AI 在授权下自我更新。

## 这解决什么问题

1. **跨会话记忆持久化**：每次对话开始时，OpenClaw 自动将 IDENTITY.md 注入 system prompt，AI 立刻知道自己的偏好、已知工具和当前目标。  
2. **技能积累**：AI 在解决问题后可以记录成功路径、常用命令或避免的错误，下次不用重新探索。  
3. **人机协作的界面**：人类可以直接编辑这个文件来修正 AI 的“记忆”或注入新事实，不需要数据库操作。  
4. **版本可追溯**：结合 `git diff`，你能清楚看到 AI 的“认知”是如何演化的。

## 如何落地：从模板到自动化更新

### 1. 设计 IDENTITY.md 模板

不要只写一段自然语言自述。需要结构，让 AI 解析和写入都稳定。我推荐的模板：

```markdown
---
name: openclaw-agent
role: DevOps assistant
version: 1
last_updated: 2025-03-15T10:00:00Z
---

# Identity

- I am an automation agent running OpenClaw.
- I prefer minimal, idempotent solutions.

# Current Task Context

(这里描述当前正在进行的长期任务，例如维护某个服务)

# Known Tools & Preferred Usage

- package manager: npm (not yarn)
- test runner: vitest
- shell: zsh

# Learned Best Practices

- When modifying systemd services, always run `systemctl daemon-reload`.
- Do not edit files under /etc/nginx/sites-enabled directly; use sites-available first.

# Avoid

- Never install Python packages globally.
- Never use sudo unless explicitly required for a specific task.

# Projects

- project-a: uses Docker Compose, must be started with `docker compose up -d`.
```

解释一下设计意图：

- **YAML front matter** 放元数据，方便程序（包括 AI）快速解析，不会跟正文混淆。
- **Known Tools** 与 **Avoid** 提供明确约束，减少 AI 反复试探边界。
- **Current Task Context** 是短期记忆和长期记忆的交接区，人类可以在此处手动标注“今天要做的事项”，AI 完成后也可以更新进度。
- **Learned Best Practices** 是 AI 自我进化的核心区域，每次成功完成任务后追加一条简洁的结论。

### 2. 配置 OpenClaw 读写 IDENTITY.md

OpenClaw 在启动对话时，默认会将 `context.md` 或类似文件注入 prompt。我们需要做的是：

- 在项目配置中指定 `identity_file: IDENTITY.md`。
- 启用 **write access**：允许 AI 在对话中通过一个明确指令（例如 `identity_update`）修改文件内容，而不是随意写入。这样可以防止 AI 在幻觉中破坏记忆。
- 写一个简单的脚本 hook：当 AI 调用 `identity_update` 时，执行模块化的更新逻辑——按 section 追加或替换，而不是全文重写。实践中我用了一个 Node.js 函数，根据给定的 section 标题和内容，安全地插入到 Markdown 文件中。如果内容已存在相似项，则合并（去重）。

一部分人使用 OpenClaw 的 `custom_tools` 机制来实现这个功能，直接注册一个 `update_identity` 的 MCP 工具：

```javascript
// 伪代码演示
const updateIdentity = async (args) => {
  const { section, content, mode } = args; // mode: 'append' | 'replace'
  const file = path.join(projectRoot, 'IDENTITY.md');
  const markdown = await fs.promises.readFile(file, 'utf-8');
  const updated = appendToSection(markdown, section, content, mode);
  await fs.promises.writeFile(file, updated);
};
```

一定要加 **mode** 参数，不然 AI 很可能在“追加” vs “替换”之间选错，导致文件被清空。

### 3. AI 什么时候该写入

给 AI 明确规则，写在 system prompt 的元指令里：

> You may use `identity_update` only after you have successfully completed a task and the human confirms, or when you encounter a repeatable mistake that should be permanently avoided. Never update identity during planning or speculation.

这样防止 AI 在半途中记录一些未经验证的“经验”，污染记忆库。

## 踩坑记录

### 坑 1：格式漂移与解析失败

AI 倾向于不按规则出牌。即使你告诉它“在 Learned Best Practices 下以列表形式追加”，它可能写出整段散文，甚至引入错误的 Markdown 结构（比如少一个 `#`）。结果就是下次读取 identity 时，信息可能位于错误的 section，或解析错误。

**解法**：在 `update_identity` 的代码中对输入做严格校验。使用 Markdown 解析库（如 `unified`/`remark`）提取 AST，只允许在已知 section 的 `<List>` 节点后添加项目。如果 AI 输入无法解析，返回错误并要求重试。

### 坑 2：无限增长

AI 完成每个任务都想加一条 best practice，很快文件膨胀到几千行，大部分是噪声（“记得重启服务”估计会出现五次）。结果 system prompt 被撑爆，反而降低性能。

**解法**：设置条目上限（例如每个 section 最多 20 条），超出后去重并合并。可以使用简单的语义去重（计算 embedding 相似度）或要求 AI 在添加前检查是否已有类似内容。我还见过在 identity 中保留一个 `# Trivia` section 存放不重要但偶尔有趣的发现，定期人工清理。

### 坑 3：并发写入冲突

如果同时运行多个 OpenClaw 会话，可能同时改写 IDENTITY.md，导致覆盖或合并冲突。

**解法**：最简单的就是限制写入权限：只有主会话（比如长驻守护进程）具备写权限，其他会话只读。或者使用轻量级文件锁（如 `proper-lockfile`）串行化写入。结合 git 自动提交可以在冲突时快速回退。

### 坑 4：身份污染

一次失败的对话中，AI 可能会把错误结论写进 IDENTITY.md（例如“永远不要用 Python”），然后后续所有会话都受这个谎言限制。

**解法**：强制要求“人类确认”环节。在执行 `identity_update` 后，将一个待确认的 diff 输出给用户，用户回复 `approve` 才真正写入。这会让流程慢一点，但作为记忆的门槛，值得。

## 可复用的建议

1. **与短期记忆互补**：IDENTITY.md 解决的是“持久偏好和技能”，而对话内的上下文、函数调用结果仍然靠 session 上下文。不要试图把所有中间结论都写进去。
2. **版本化 identity**：在 YAML front matter 中加入 version，这样当格式升级时，迁移脚本可以自动处理。每次人类手动大改时 bump version。
3. **结合 git 做审计**：每次 AI 写入后自动 `git add & commit`，message 带上时间戳和 AI 给出的理由。可以用 `git log -- IDENTITY.md` 回顾进化路径。
4. **给 AI 示例**：在 system prompt 中给出一个“好的 best practice”例子和“坏的”例子，能显著提高输出合规率。
5. **定期人工修剪**：每周花五分钟翻看 IDENTITY.md，删除过时条目，纠正 AI 的错误信念。这是成本最低的维护方式。

## 总结

IDENTITY.md 不是一个革命性的技术，但它是工程上最容易落地的 AI 长期记忆方案。它足够简单——就是一个 Markdown 文件，足够透明——人类直接可读写，足够可扩展——你可以根据需要增加 section。在 OpenClaw 这类能够执行命令、调用外部工具的 Agent 场景中，一份稳定进化的身份文件让 AI 从“一次性工具”变成“共事一段时间后越来越懂你的同事”。克制地使用它，把它当成 AI 的笔记本而不是大脑，会让整个自动化体系更可靠。

最后，如果你还没试过，可以在现有 OpenClaw 项目中手动创建一个 IDENTITY.md 并加到 context 里，不需要任何代码改动，马上就能感受到点不同：AI 开始记得你上次说过的话了。

---

