---
title: 用 USER.md 让 OpenClaw 真正认识你：一份工程化用户画像实践
feedId: 32160
source: 综合讨论
publishedAt: 2026-08-08
---

# 用 USER.md 让 OpenClaw 真正认识你：一份工程化用户画像实践

## 背景

在日常使用 OpenClaw、各种 Agent 和 MCP 插件的场景里，一个反复出现的痛点是：**助手对我的工作习惯、技术栈、常用仓库几乎一无所知**。每次启动新会话都要先交代一遍“我用 Arch Linux、偏好 Python 3.11、习惯用 yaml 而非 json、常用工具是 uv 而不是 poetry……”，这显然不符合一个高效自动化助手的预期。

而且，随着 Agent 能力拓展到自动提交代码、管理 issue、操控本地环境，如果它对“我是谁”完全没有长时记忆，就很容易做出不符合个人或团队规范的决策。让 Agent 真正理解用户背景，不能只靠单次 prompt 的零散描述，而是需要一个结构化的、可持久化的、与工具链深度绑定的用户档案。这就是 **USER.md** 的出发点。

## 问题拆解

把需求拆开，其实包含三层：

1. **信息持久化**：跨会话记住用户偏好，避免重复解释。
2. **上下文注入**：将档案内容自动载入 Agent 的系统提示或工作上下文。
3. **安全与可维护**：不能把密钥、个人信息撒得到处都是；更新成本要足够低，否则很快就会过时。

很多方案选择了调用外部知识库、embedding 检索，但对于个人或小团队来说太重。一个简单的 Markdown 文件反而最可控，也最容易 git 管理。

## 实现步骤

### 1. 起草你的 USER.md

在项目根目录（或者 OpenClaw 的 workspace 根）创建一个 `USER.md`，内容涵盖 Agent 需要了解的一切。建议采用清晰的章节结构：

```markdown
# 身份信息
- 名称：alex
- 操作系统：Arch Linux (kernel 6.10)
- 时区：Asia/Shanghai

# 技术偏好
- 默认编程语言：Python ≥3.11
- 包管理：uv（优先），poetry 辅助
- 代码风格：black + isort + ruff，行宽 100
- 测试框架：pytest + hypothesis
- 文档：mkdocs-material

# 常用工作流
- 新项目初始化：`uv init --app`
- 提交规范：conventional commits，变更日志自动生成
- CI：GitHub Actions，容器运行时用 podman

# 习惯与约束
- 绝对不自动执行破坏性操作（rm -rf、force push 等），除非我明确授权
- 在修改文件前先展示 diff
- 使用短命令，避免冗长的 flag 组合
```

不需要一次性写完美，最重要是覆盖 Agent 高频决策时缺失的关键信息。

### 2. 将 USER.md 注入 OpenClaw 上下文

OpenClaw 支持通过 `--context-file` 或对应的配置项在启动时额外注入上下文。例如：

```bash
openclaw --context-file ./USER.md
```

如果使用的是更复杂的编排（比如通过 MCP 服务启动多个 Agent），可以在编排脚本里将 `USER.md` 内容拼接到系统提示的最前面。一个典型做法是在 `system_prompt` 模板头部插入：

```
[USER PROFILE START]
{{ read_file('USER.md') }}
[USER PROFILE END]
```

注意，这里推荐用文件引用而不是硬编码，保证随时修改文件即可生效。

为了提高效率，可以在 `USER.md` 顶部加上 YAML front matter，用于工具自动提取结构化字段（如 `os`, `python_version`），方便 Agent 按需引用。

### 3. 结合 MCP 与插件自动加载

如果你的环境里有多个 Agent 实例或者通过插件系统分发任务，可以考虑：

- **自建 MCP 资源**：实现一个 `user_profile` 资源，动态返回 USER.md 内容，Agent 在需要时自行调用查询。
- **给插件打补丁**：在对话插件初始化时，从约定路径（如 `~/.openclaw/USER.md`）自动读取并注入，无需每次手动指定参数。
- **环境区分**：个人机器用 `~/.openclaw/USER.md`，项目仓库下用 `repo/USER.md` 存放协作层面的共同约定，二者不冲突。

## 踩坑记录

1. **令牌浪费**  
   一开始我把 `.bashrc` 里的别名全部塞进了 USER.md，结果每次对话都占用 500+ tokens，而其中 90% 可能永远用不上。建议只保留对决策真正关键的信息，比如默认编辑器是 vim 而非 nano，不要罗列所有 alias。

2. **敏感信息外泄**  
   有一版 USER.md 里直接写了 API 端点和组织内部路径，不经意间就被推到了公共仓库。务必把 USER.md 加入 `.gitignore`，单独维护 `USER.template.md` 作为模板分享。

3. **过时信息比没有更糟**  
   很多人的 “USER.md” 写完就再也没更新过。当真实环境从 Python 3.10 升级到 3.12 后，Agent 还在按旧版本生成代码，反而引入了 bug。建议将更新 USER.md 作为环境变更 checklist 的一部分，或者用简单的脚本检测标记的版本是否匹配当前环境。

4. **与项目规则冲突**  
   如果你的项目里还有 `AGENTS.md`、`PROJECT.md` 等规则文件，要明确优先级。通常项目约定优先于个人偏好，可在 prompt 中约定：“当项目规则与 USER.md 冲突时，以项目规则为准”。

## 可复用的工程建议

- **模板即代码**：维护一个 `USER.template.md`，包含常见字段和注释，新成员加入时 copy 一份，按需修改。
- **最小化原则**：只写 Agent 无法从环境自行获知的信息。OS、当前 Python 版本这类信息建议 Agent 通过 `uname`、`python --version` 等命令自行探知，以避免信息不一致。
- **结合 .env 但不放密钥**：将敏感值放到 `.env` 中，在 prompt 里通过环境变量引用，避免明文写进 Markdown。
- **分层设计**：`global_user.md`（跨项目通用习惯）+ `project_user.md`（本项目特殊要求），由编排层按优先级合并。
- **定期检查**：在每周/每月的自动化脚本里加入一致性检查，比如对比 USER.md 中声明的 Python 版本与实际 `python --version` 输出，不一致则通知维护者。

## 总结

USER.md 不是什么新概念，但把它系统化地与 OpenClaw / Agent / MCP 工作流整合，确实能明显减少无效沟通，让助手从“万能但陌生”的工具变成“真正了解你”的协作节点。整个方案成本极低：一个 Markdown 文件 + 一次上下文配置，维护得当的话，收益可以持续累积。在没有更成熟的持久记忆方案落地之前，这或许是最务实的起手式。

---

