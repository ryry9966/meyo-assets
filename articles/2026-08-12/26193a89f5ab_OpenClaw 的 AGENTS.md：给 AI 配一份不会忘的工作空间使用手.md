---
title: OpenClaw 的 AGENTS.md：给 AI 配一份不会忘的工作空间使用手册
feedId: 32638
source: 综合讨论
publishedAt: 2026-08-12
---

# OpenClaw 的 AGENTS.md：给 AI 配一份不会忘的工作空间使用手册

## 背景：代理为什么需要一张“说明书”

OpenClaw 这类 AI 编程代理，通过 MCP 插件可以读写文件、执行命令、调用外部工具。它们能力很强，但有一个根子上的毛病：**每次新会话，它们对项目约定一无所知**。你上次花 20 分钟教会它“测试必须用 `pytest --strict-markers`”，下一次打开新窗口，一切归零。

AGENTS.md 就是为此设计的。它是一份放在项目根目录下的 Markdown 文件，OpenClaw 启动时会自动读取并注入到系统上下文，成为代理的“工作空间使用手册”。它不替代 MCP 工具配置，也不接管编译器，只是安静地告诉代理：这间房子的插座在哪、开关怎么用、哪些地板不能踩。

## 问题：没有规则，代理的自由会让你头疼

第一次用 OpenClaw 处理中等规模的 repo 时，多数人会遇到这些情况：

- 代理把临时脚本写在根目录，命名毫无规律，`fix.py`、`fix2.py`、`test_fix3.py` 满天飞。
- 跳过你配置好的 lint 命令，直接用 `python` 运行，结果 CI 红了一片。
- 擅自修改 `.github/` 下的 workflow，因为它觉得“可以优化”，但你没让它碰 CI。
- 在不同任务里，反复使用 `npm` 而不是你项目要求的 `pnpm`，导致 lockfile 冲突。

本质原因不是模型笨，而是**没有边界信息**。AGENTS.md 就是在给代理画边界。

## 做法：为 OpenClaw 配置 AGENTS.md 的步骤

### 1. 确定存放位置和加载方式

OpenClaw 支持两种路径：

- `./AGENTS.md` （项目根目录，推荐）
- `./.agents/AGENTS.md` （如果你希望把代理配置都收进一个目录）

在 `openclaw.yaml` 中显式启用（有些版本默认开启，但最好确认）：

```yaml
agents:
  instruction_file: AGENTS.md   # 或 .agents/AGENTS.md
  injection_level: system       # system | user ，推荐 system
```

如果你同时使用 MCP 扩展，AGENTS.md 的加载顺序在工具描述之后，系统提示的最外层，优先级足够高。

### 2. 写一份“可执行”的规则，而不是一篇文章

避免长篇散文。代理擅长理解结构化、命令式的短句。我推荐的写法是按功能分块，每块用明确的“必须/禁止”开头。

示例：

```markdown
# AGENTS.md

## Project context
- Language: Python 3.11+, package manager: pdm
- Test runner: pytest with --strict-markers
- Linting: ruff, mypy (strict mode)

## Directory conventions
- All scripts MUST be placed under `scripts/` with a descriptive name.
- Temporary files MUST go into `data/.tmp/`. Never create files in root.
- Do NOT edit anything under `docs/` unless the task explicitly mentions documentation.

## Tool usage
- Always use `pdm run` to execute tools: `pdm run pytest`, `pdm run ruff check .`
- Do NOT use `pip install`; dependencies are managed by pdm.

## Constraints
- Never modify CI workflows in `.github/workflows/`.
- Do not commit without asking, unless the task says "commit".
- When in doubt, ask for clarification instead of assuming a file can be deleted.
```

记住一条原则：**正向表述 > 否定表述**。  
“Always place generated scripts into `scripts/`” 比 “Do not create scripts anywhere else” 更容易让代理遵循。

### 3. 验证代理是否真的在读

写好 AGENTS.md 之后，用一个简单的探针任务测试：

> “Create a small helper script. Where will you put it, and how will you run the tests?”

如果代理回答 “I'll place it under `scripts/` and use `pdm run pytest`”，你的 AGENTS.md 生效了。如果它开始列出根目录路径、用 `python -m pytest`，说明未加载或被忽略。

## 踩坑点

1. **文件名大小写或扩展名错误**  
   必须是 `AGENTS.md`，不是 `agents.md` 或 `AGENTS.MD`。某些文件系统（如 macOS 默认）不区分大小写，本地测试没问题，推到 Linux CI 立刻失效。统一用大写。

2. **规则太抽象，缺乏可操作细节**  
   “请保持代码整洁”这类话对代理无用。要具体到命令、目录、文件匹配模式。

3. **上下文窗口被 AGENTS.md 占满**  
   如果你的 AGENTS.md 超过 150 行，相当于预消耗了系统提示中的大块 token。推荐核心规则在 60 行以内，详细参考资料（如完整的 API 规范）拆分成 `docs/agent-ref.md`，让代理在需要时用工具读取。

4. **与其他约定文件的冲突**  
   项目里可能已经有 `.cursorrules`、`.github/copilot-instructions.md`。OpenClaw 不会合并它们，只认 AGENTS.md。建议团队商量好，统一使用 AGENTS.md，并让其他文件通过 `include` 引用（比如在 Copilot 设置里指向同一份文件）。

5. **代理“过于听话”导致死循环**  
   比如你写了 “Never delete any file”，但某次任务确实需要清理过期缓存，代理会选择卡住而不是请求例外。正确做法是写成 “Do not delete files unless the task explicitly requires cleaning specific temporary paths.”

## 可复用建议

- **使用模板快速起步**  
  在团队或开源项目里维护一份 AGENTS.md 模板，包含：Project Overview / Toolchain / Do’s / Don’ts / Examples。新人启动 OpenClaw 时直接复制修改。

- **按作用域分块标记**  
  如果你同时用不同 MCP 插件，可以用 HTML 注释做标记，便于未来做自动化裁剪：
  ```markdown
  <!-- AGENT:FILE_OPS -->
  All file edits must use `write_file` tool only after reading the target.
  <!-- /AGENT:FILE_OPS -->
  ```

- **版本化并纳入 code review**  
  AGENTS.md 的变更应该走 PR，因为它是团队对代理行为的共同约定。代码审查时也留意代理生成的内容是否违背了这份手册。

- **结合 MCP 工具做自检**  
  可以写一个简单的 MCP 工具，在任务结束后检查代理是否在禁止目录下留下了文件、是否使用了禁用命令。如果违规，给出警告并让代理修正。这种闭合回路让规则更强。

## 总结

AGENTS.md 不是“银弹”，它不会让代理变聪明，但能让它变规矩。工程团队花几十分钟写好这份手册，后续每次 OpenClaw 会话都能省下大量对齐时间。它本质上是一份**给 AI 的团队入职文档**——你可以把它想象成给一个新人工程师的“日清日结”工作说明，写得越具体、越可验证，代理就越不容易在关键路径上放飞自我。

如果你还没有开始用 AGENTS.md，那下一次启动 OpenClaw 时，先别急着给任务，先把规矩立起来。

---

