---
title: 从 AGENTS.md 看懂 OpenClaw 的工作空间约束：让 Agent 像工程师一样操作文件
feedId: 32177
source: 综合讨论
publishedAt: 2026-08-09
---

## 为什么需要 AGENTS.md

在 OpenClaw 这类面向 Agent 的工作平台里，MCP（Model Context Protocol）给了模型调用文件系统、版本控制、Shell 等外部工具的能力。便利的另一面是，Agent 很容易“过度自由”——在项目根目录下到处写临时脚本、无视约定的输出路径、甚至覆盖已有代码。System prompt 能兜底一些行为，但分散、不易版本化，而且换一个 workspace 就要重新适配。

工程团队真正需要的是一个**可维护、可评审、跟着仓库走的规则文件**，让 AI 在进入工作区之前就明确边界。这就是 AGENTS.md 出现的原因。它相当于给 AI 的工作空间使用手册，描述了“在这个目录下，你能做什么、应该怎么做、以及有哪些模板和工具可用”。

## 问题场景

我们团队在多个 OpenClaw workspace 中复用同一个 Agent，但每个项目的目录结构、工具链、输出格式都不同。早期只靠口头约定，经常出现：

- 数据提取脚本被写到 `src/` 里，污染了源码目录；
- Agent 生成的报告名不一致，二次处理脚本抓不到文件；
- MCP 提供的工具虽然多，但 Agent 有时候使用了不该用的命令（比如 `delete_file`）；
- 切换项目后，Agent 仍沿用上一个 workspace 的习惯。

这些问题本质上不是模型能力不足，而是**缺少工作空间级别的行为契约**。AGENTS.md 就是那纸契约。

## 实现做法

在 OpenClaw 工作区的根目录放置一个 `AGENTS.md` 文件，平台会在启动 Agent 时自动将其内容注入到系统指令中（通常是拼接到 MCP 工具列表之前或之后）。文件本身以 Markdown 编写，我们总结下来的有效结构如下：

```markdown
# Workspace Rules
- 所有输出文件必须放在 `output/` 目录下，按 `YYYY-MM-DD` 日期子目录归档。
- 生成代码前必须先阅读 `src/` 下对应的模块，避免重复。
- 禁止在项目根目录创建任何 `.py`、`.sh`、`.json` 临时文件。

## Allowed Tools
只允许使用以下 MCP 工具（其他调用会被平台拒绝）：
- read_file
- write_file
- search_code
- execute_command (仅限 `make format`, `make test` 等预定义命令)

## Output Format
生成的报告必须使用 `templates/report.md` 模板，填充字段后保存为 `output/report_<task_id>.md`。

## Context Hints
- 本项目主要语言是 Python 3.11，所有脚本默认使用该版本。
- 数据库连接配置在 `config/db.yaml`，不要硬编码。
```

这份文件不复杂，但一旦被 Agent 读取，它的行为会明显“收敛”。我们在调试面板中看到，Agent 会主动引用 AGENTS.md 里的规则来决定工具选择，比如自动跳过 `delete_file`，或者在生成文件前先读取模板。

接入步骤很简单：

1. 在项目根目录创建 `AGENTS.md`，根据团队规范填充规则。
2. 确保 OpenClaw 客户端/CLI 配置中打开了“加载工作空间规则”开关（默认开启）。
3. 用一个小任务测试，观察 `Agent trace` 或日志，确认规则被成功注入并生效。
4. 将 `AGENTS.md` 提交到 Git，随项目版本一起迭代。

## 踩坑记录

**坑一：工具白名单与实际 MCP 名称不一致**  
MCP server 发布的工具名可能带前缀（例如 `mcp__io_write_file` 而非 `write_file`）。AGENTS.md 里写了别名或不完整名称，会导致 Agent 认为该工具不可用，行为异常。解决办法是在 OpenClaw 的工具面板中确认完整名称后再写入文件。

**坑二：路径约束过于绝对**  
比如强制所有输出到 `output/`，但 Agent 如果本身就需要创建中间文件，会陷入无目录可用的尴尬。建议在规则中留出例外：“允许在 `.temp/` 下创建临时文件，任务结束后由 Agent 自行清理”。

**坑三：文件长度挤占上下文**  
AGENTS.md 写得过于详细（比如贴上完整模板、大段代码示例），会占用模型本就有限的上下文窗口。我们的做法是只写规则，模板文件用相对路径引用，让 Agent 需要时再用 `read_file` 去读取。

**坑四：规则失效时的静默异常**  
如果 AGENTS.md 中存在语法歧义，Agent 可能忽略整段而退回到默认行为，但不会报错。建议定期做“合规性抽查”：手动给一个违反规则的指令，看 Agent 是否拒绝或纠正。

## 可复用建议

- **模板化初始化**：把 AGENTS.md 模板放进团队脚手架，新项目在执行 `openclaw init` 时自动生成空壳文件。
- **分层规则**：如果多个 workspace 共享基础规范，可以把通用部分写成 `base_rules.md`，在 AGENTS.md 中用 `import` 或注释方式手动引用（目前 OpenClaw 还不会自动 include，但人类维护者知道怎么合并）。
- **结合 pre-commit 做事后校验**：在 Git hooks 中增加检查脚本，扫描 Agent 生成的提交里是否有违反 AGENTS.md 的行为（比如根目录下出现脚本），一旦发现就打回。这补充了 AI 遵守规则但偶尔“越界”的薄弱环节。
- **把 AGENTS.md 当成团队协作产物**：规则不是只写给 AI 看的，人类同事也能通过它快速理解工作空间的自动化流程，降低新人上手成本。

## 总结

AGENTS.md 并非什么创新概念，它更像是传统工程里 `CONTRIBUTING.md` 或 `Makefile` 的 AI 适配版。在 OpenClaw 平台上，它通过拼入系统指令，充当了 Agent 的工作空间使用手册。花十几分钟写好这个文件，换来的是跨项目一致的 Agent 行为、更少的意外操作、以及一套可审查的自动化边界。如果你们已经开始在生产环境里用 Agent 直接操作仓库，那么 AGENTS.md 是比“在聊天里提醒”更可靠的契约。

---

