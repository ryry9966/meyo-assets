---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35478
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw / Agent 实践里，`tools.md` 逐渐成为本地配置的一个入口。它并不是强制标准文件，但在多机器、多项目、多自动化任务的场景下，用一份 Markdown 描述本机环境、命令路径、目录约定，能明显减少 Agent 反复试探和猜错命令的时间。

如果本地环境差异只靠任务 prompt 动态描述，每次都要重复说明，而且容易遗漏。`tools.md` 的作用是把这些差异显式化、结构化，成为 Agent 读取的“首个环境上下文”。

## 问题

同一个 Agent 任务，在一台机器上可以正常执行，换到另一台机器上就失败，常见原因包括：

- 本地 Python 环境不同，有人用 `python`，有人用 `python3`；
- 项目依赖路径不一致，`venv` 有的在 `.venv`，有的在 `venv`；
- 本地工具命令不同，例如 `code` 没有配置到 PATH；
- API 密钥、数据库地址等配置散落在不同文件里，Agent 读取不到；
- Windows 和 macOS/Linux 的命令差异被写死在 prompt 里。

## 做法 / 步骤

### 1. 先定义 tools.md 的层级

建议区分三个层级：

- 全局：放在用户主目录，描述机器级别的通用工具和路径；
- 项目级：放在项目根目录，描述该项目专用环境；
- 会话级：在对话中临时给出的覆盖项，优先级最高。

Agent 在启动任务前先读全局和项目级 `tools.md`，再叠加会话内的指令。这样可以避免“只改一处就影响所有任务”。

### 2. 用表格或键值块描述关键项

内容不需要复杂，但要覆盖几类信息：

- 环境变量：如 `OPENAI_API_KEY` 是否已设置、放在哪个文件；
- 命令别名：如 `python` -> `python3.11`；
- 路径别名：如项目数据目录、缓存目录；
- 工具开关：哪些 MCP 服务可用，哪些插件默认关闭；
- 平台差异：如 macOS 用 `brew`，Linux 用 `apt`。

示例结构：

```markdown
## 本机环境
- OS: macOS
- Shell: zsh
- Python: python3.11 (via brew)
- Node: 20.11

## 命令别名
- python -> python3.11
- pytest -> poetry run pytest

## 目录约定
- 缓存目录: ~/.cache/myagent
- 输出目录: /tmp/agent-output

## 服务与插件
- mcp-server-filesystem: enabled
- browser-plugin: disabled
```

### 3. 与 .env 和 secrets 分离

`tools.md` 里不要写真实密钥。只写“密钥来源”，例如 `API_KEY 从项目根目录 .env 读取`，或 `数据库连接串由环境变量 DATABASE_URL 提供`。真正的值放 `.env` 或系统密钥管理里。这样 `tools.md` 可以提交到版本库，而 secrets 仍然被忽略。

### 4. 让 Agent 自动校验不一致

可以给 Agent 一个轻量校验脚本，在开始任务前对比 `tools.md` 声明和实际环境。例如检查 `python --version` 是否匹配，检查声明的路径是否存在。如果不一致，先输出差异列表，再决定是否继续任务。不要盲目相信 `tools.md` 是“最新”的。

## 踩坑点

- **把 tools.md 当成万能配置**：它适合描述“差异”和“入口”，不适合把所有业务逻辑塞进去，否则会变成另一个需要维护的代码库。
- **硬编码绝对路径**：`/Users/alice/projects/...` 在别人机器上直接失效。优先使用 `~`、环境变量或相对项目根目录。
- **secrets 明文混入**：一旦提交到 Git，就不好撤回。用 `.env.example` 替代或只写读取方式。
- **平台差异只写一半**：例如写了 Linux 的 `apt install`，没写 macOS 的 `brew install`。Agent 在另一平台会继续踩坑。
- **tools.md 与实际环境漂移**：环境升级后没有更新，Agent 读取到旧版本会做出错误操作。建议在 CI 或定时任务里做一次环境快照比对。

## 可复用建议

1. **模板化**：建一个 `tools.template.md`，新项目直接复制，强制包含环境变量、命令别名、目录约定、服务开关四块。
2. **用脚本生成**：写一个 `detect_env.sh` 或 Python 脚本，自动探测本机 Python、Node、路径，输出初始 `tools.md`，人工再微调。
3. **版本控制**：项目级 `tools.md` 跟随代码版本管理，全局 `tools.md` 放在私有 dotfiles 仓库。
4. **读写权限拆分**：Agent 对 `tools.md` 默认只读；需要修改时必须走用户确认，防止 Agent 自我修改导致不可追溯。
5. **与 MCP 集成**：如果使用 MCP，可以让一个 `local-config` 工具专门读取 `tools.md`，返回结构化数据，避免 Agent 自己解析 Markdown 出错。

## 总结

`tools.md` 不是银弹，但它能显著减少本地环境差异带来的 Agent 失败。核心原则是：显式描述差异、分层管理、secrets 分离、定期校验。与其让 Agent 在每次任务里猜测你的本机环境，不如把已知差异写清楚，让自动化从确定性的起点开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6b65f3d53a3bf7cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b66c1fee26e31eee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ede5c182d93d09cd.png)

