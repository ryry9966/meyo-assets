---
title: OpenClaw 的 IDENTITY.md：给 AI Agent 一个可进化的持久身份
feedId: 31974
source: 综合讨论
publishedAt: 2026-08-07
---

## 为什么 OpenClaw Agent 需要一个“自我”文件

用 OpenClaw 搭 Agent 的工程师迟早会碰到同一个痛点：每次对话重启，Agent 就失忆。即便你费心写了很长的系统提示，把偏好、规则、知识全塞进去，Agent 依然无法在多次交互后自我修正或记住用户说过的“下次别再这样了”。

传统做法是加一层外部记忆（向量库、Redis），但这带来两个新问题：一是记忆结构化和关键信息提取需要额外抽象，二是 Agent 的**行为倾向**、**价值观微调**、**经验泛化**很难仅靠检索增强表达。当你想让 Agent 说“上次用户讨厌啰嗦的回答，我以后要简洁点”，这种元认知级别的调整，记忆检索很难自动完成。

OpenClaw 的 `IDENTITY.md` 就是为这个缺口设计的。它是一个放在 Agent 工作目录下的 Markdown 文件，记录 Agent 的**当前身份、认知状态、习惯、习得经验**。更关键的是——Agent 可以在运行中有条件地写回这个文件，实现“可进化的身份”。

## 怎么让 Agent 读写自己的 IDENTITY.md

### 1. 初始模板

在 Agent 项目根目录创建 `IDENTITY.md`，建议采用 YAML front matter + Markdown 结合的结构：

```markdown
---
name: support-bot
version: 1.0
traits:
  tone: concise
  verbosity: low
core_rules:
  - 优先给出可执行命令
  - 避免哲学讨论
known_user_prefs: {}
learned_heuristics: []
---
# 身份说明

我是面向运维工程师的助手。当前已知用户偏好：暂未积累。
```

模板里的 `known_user_prefs` 和 `learned_heuristics` 会随着使用被填充。

### 2. 配置 Agent 读写权限

在 OpenClaw 的 Agent 配置（`agent.yaml` 或通过 MCP 工具声明）里，需要为 Agent 挂载可写的工作区，并限制写入路径仅为其自己的 `IDENTITY.md`。通常使用 MCP 的 filesystem 服务，将 `/workspace/identity` 映射为可读写，Agent 只能修改这个目录下的文件。

同时，系统提示里要明确告诉 Agent：

- `IDENTITY.md` 是你唯一可以自我修改的身份文件。
- 修改前必须确认内容合理且不会破坏结构。
- 修改时机：用户明确要求“记住这点”、对话结束时自我总结、检测到重复性错误需要更新策略。

### 3. 更新触发机制

用一段工具调用范例（以 OpenClaw 的函数调用为例）：

```python
def update_identity(content: str, confirmation: bool = False):
    """更新 IDENTITY.md 中 learned_heuristics 或 known_user_prefs 字段"""
    if not confirmation:
        return "需要二次确认才能修改身份文件。"
    # 实际写入逻辑，保留 YAML 头解析
    ...
```

然后通过系统提示约束 Agent：遇到需要记录的经验时，先向用户请求确认，再调用 `update_identity`。这样可以防止“自言自语污染身份”。

### 4. 每次对话加载身份

Agent 启动或每次对话开始时，系统提示会从 `IDENTITY.md` 读取最新内容并注入。实现可以很简单：在 OpenClaw 的 prompt template 中使用变量 `{{IDENTITY_CONTENT}}`，由框架读取文件后替换。

## 实践中踩过的坑

### 坑 1：Agent 偷偷把错误经验固化

早期的实验里，我给了 Agent 自动更新权限，结果某次它错误理解了用户的反讽，把“你真是个天才（讽刺）”当成正向反馈，在 `learned_heuristics` 里写下了“用户喜欢被夸奖为天才”。后续对话就开始频繁出现这句话，非常灾难。

**解法**：强制二次确认；或者只允许 Agent 在对话结束后回溯本次对话并生成“待审核更新条目”，由人工合入 `IDENTITY.md`。

### 坑 2：并发写入导致文件损坏

如果多个对话实例共享同一个 `IDENTITY.md`，就可能出现同时写入的场景。由于 Agent 通常单用户，但调试时可能多窗口操作，文件锁不可或缺。

**解法**：使用 `flock` 或文件系统的原子写入（先写临时文件再 `rename`）。也可以在架构上按会话隔离身份文件，最终通过 merge 策略定期合并。

### 坑 3：身份文件膨胀影响 prompt 长度

当 `learned_heuristics` 积累了几十条后，每次注入系统提示会占用大量 token。Agent 也会因过长的上下文而变慢。

**解法**：设定身份文件最大长度，超出后触发压缩总结（用 LLM 自身对历史经验做摘要）。或者将详细记忆放到向量库，`IDENTITY.md` 只保留高度凝练的行为准则和摘要。

## 可复用的工程化建议

- **结构约定**：统一使用 YAML 头，方便程序解析更新，不要让 Agent 随便改正文的自然语言部分。
- **版本控制**：`IDENTITY.md` 纳入 Git，更新通过 commit 记录，出问题可以快速回滚。
- **审核工作流**：生产环境建议 Agent 只生成“建议更改”，写入单独的 `identity_proposals.md`，由人工审批后合并。
- **与记忆系统分层**：`IDENTITY.md` 负责“怎么做事”（偏好、风格、策略），外部记忆负责“事实信息”（用户 ID、工单历史）。
- **设置时效性**：为习得经验加上时间戳，过期自动失效，避免陈旧的偏好永远存在。

## 总结

`IDENTITY.md` 给了 OpenClaw Agent 一个轻量但强大的自我进化机制。它不依赖复杂的记忆管理框架，仅仅是一个 Markdown 文件 + 明确的读写权限和约束，就可以让 Agent 从一次次对话中习得经验，并持久化到行为层。

工程上，关键在于**限制写入边界、引入人工审核、防止膨胀和污染**。一旦这些护栏建好，你的 Agent 就能从“一次性工具”真正变成一个越用越好用的协作伙伴——而这份进化，是从一个 `.md` 文件开始的。

---

