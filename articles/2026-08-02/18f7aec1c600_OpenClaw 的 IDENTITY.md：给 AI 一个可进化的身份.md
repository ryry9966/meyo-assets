---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 31262
source: 综合讨论
publishedAt: 2026-08-02
---

## 为什么需要一份“活”的身份文件

用 OpenClaw 搭过 Agent 的人都有体会：系统提示写得再详细，对话长了以后模型还是容易“忘掉”自己的角色设定。更麻烦的是，有些信息本该随着使用持续积累——比如用户的命名习惯、项目目录结构、常用工具的偏好参数——硬编码在提示里既不灵活，也不好维护。

OpenClaw 在 workspace 里留了一个特殊文件 `IDENTITY.md`，不是简单的小说式设定，它更像一个可被 Agent 自身读写的“身份数据库”。配合 filesystem MCP 和 OpenClaw 的 introspection 能力，我们可以让 Agent 在任务过程中主动更新这个文件，把有价值的知识沉淀回身份里，真正做到“越用越懂你”。

这篇帖子会拆解这个模式的工程落点，从背景、实现步骤到踩坑和复用，都尽量不神话，只说真实可跑的东西。

## 核心思路：区分静态身份与动态知识

传统的 Agent 身份大概长这样：

```
You are a senior backend engineer. You prefer Python and Rust. 
Always use `pnpm` as the package manager...
```

这段文本粘在系统提示的前面，每次对话都全量传入，但模型并不会自动把对话中学到的新事实写回这个文件。OpenClaw 的 IDENTITY.md 好用的地方在于它本身是一个 **workspace 文件**，可以走标准 MCP 文件系统接口被 Agent 读写。这样一来，身份的更新就是一次文件操作，不再依赖记忆插件或外部数据库的额外接入。

通常我会把 `IDENTITY.md` 分成两个区：

- **静态区（不可变）**：核心角色定义、安全约束、工具使用规则等，由人工维护。
- **动态区（可进化）**：用户偏好、项目上下文、已知的工作流快捷方式、常见陷阱等，由 Agent 在获得授权后自动追加或替换。

二者用清晰的分隔标记切分，例如 `<!-- DYNAMIC -->` 注释，方便解析和追加。

## 实现步骤：让 Agent 自己能“记住”

1. **准备 workspace 与 MCP 权限**  
   确保 OpenClaw 的 workspace 里已经有 `IDENTITY.md`，并且在对应的 agent 配置中挂载了 filesystem MCP 服务，允许读写该文件。建议只暴露 workspace 目录，避免越权风险。

2. **写一个初始模板**  
   初始的 `IDENTITY.md` 里除了基础角色描述，还要预留动态区，甚至预先写好一个“更新指引”，比如：
   ```
   <!-- DYNAMIC -->
   ## Learned Preferences  
   (will be filled by me during tasks)
   ```
   这样 Agent 看到空结构时，更容易理解如何填充合法内容。

3. **给 Agent 添加自省更新指令**  
   在系统提示（或 IDENTITY.md 的静态区）里增加一段可操作的语言，类似：
   ```
   When the user explicitly asks you to remember something, 
   or when you complete a task that reveals a lasting preference/tool convention, 
   propose an update to the dynamic section of IDENTITY.md. 
   Use the filesystem MCP to read, edit, and write the file. 
   Always show the diff before writing and wait for user confirmation.
   ```
   这里最关键的是 **先提出 diff 并等候确认**，防止 Agent 自作主张污染身份文件。

4. **测试触发闭环**  
   给一个具体任务，比如：“请帮我把这个项目的构建命令统一改成 `pnpm build`，以后都这么用。” Agent 执行完后，应该能主动读取 `IDENTITY.md`，生成一段 Markdown diff 展示要添加的动态条目，再等你确认后通过 filesystem MCP 写入。

5. **引入版本控制**  
   一旦身份文件开始被 Agent 修改，`git init` 就有必要了。每次写入自动 commit，方便回溯哪次更新导致 Agent 行为跑偏。也可以在确认前把待写入的内容存进临时分支，人工 merge。

## 踩坑与排障

- **格式污染**：Agent 可能会在追加动态内容时破坏 Markdown 结构，比如多加一层 `##` 导致层级错乱。建议在系统提示里给出明确模板，甚至用 JSON block 来存动态信息，更稳妥。
- **频控缺失**：如果不加控制，Agent 可能每次对话都提议更新，打扰使用者。定好触发条件（如仅当用户说“记住”或任务里程碑）可以有效减少噪音。
- **权限过大引发风险**：直接给 Agent 整个 workspace 的写权限容易出问题。严格限定文件白名单，只允许访问 `IDENTITY.md` 和少数日志文件。
- **身份漂移**：Agent 有时会把错误信息当成知识记进去，比如理解错的用户偏好。用 diff 审阅和 git history 可以及时发现，必要时回滚。
- **模型差异**：部分模型对文件系统操作指令的遵循度不够高，可能需要更详细的工具描述或调整 prompt。先在小范围对话里验证稳定性。

## 可复用建议

1. **用注释标记静态/动态边界**  
   推荐 `<!-- STATIC -->` 和 `<!-- DYNAMIC -->` 成对出现，让脚本或 Agent 一目了然。

2. **动态信息结构化**  
   不必全用自然语言散文，可以用固定键值对或 YAML front matter 存偏好，便于未来程序化读取。例如：
   ```
   ---
   preferred_package_manager: pnpm
   default_shell: zsh
   ---
   ```

3. **设计更新审批工作流**  
   可以考虑在 OpenClaw 里加一个 `confirm-update` 指令，或者利用 Git 的 pull request 流程，让 Agent 创建分支、提交修改后由用户手动 merge。

4. **与笔记系统联动**  
   如果项目中还用了类似 `NOTES.md` 或 `Knowledge Base` 文件夹，`IDENTITY.md` 可以只存“关于对话应该如何发生”的身份规则，把纯粹的领域知识放在 `KNOWLEDGE.md` 里做分割，避免单一文件膨胀。

5. **周期性复盘**  
   设定一个定期任务，让 Agent 回顾近期对话，总结新的长期偏好并提议更新身份文件。这就像一个人定期反思并更新自己的“使用说明书”。

## 总结

OpenClaw 的 `IDENTITY.md` 把“身份”从一个写死的 prompt 片段，变成了一个活的、可以随着每一次交互进化的实体。不依赖外部数据库，不需要复杂的记忆引擎，只要 Agent 能操作本地文件，就能实现身份的持续沉淀。踩坑点集中在格式控制、权限和更新节奏上，这些用简单的规范就能防住。

在长期使用 Agent 的场景里，这种“自我进化”的能力比任何花哨的 feature 都更接近工程意义上的可靠。你可以从一个泛用的助手开始，让它慢慢长成最熟悉你代码库和偏好的那个专属合作者。做到这一点，代码不超过两百行文件，依赖只多了一个 Git。

---

