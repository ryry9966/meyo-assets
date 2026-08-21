---
title: Agent 的 tools.md：把本地环境差异变成可执行事实
feedId: 34030
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw/Agent/MCP/插件自动化实践里，经常出现一种情况：同一个任务在 A 机器能跑通，换到 B 机器就不行；Agent 在容器里能定位到 Python，在 macOS 上却默认调用系统 Python；插件明明配置了，MCP 工具列表里却看不到。多数时候不是 Agent 能力不够，而是它拿到的环境信息是“通用假设”。

tools.md 的价值就在这里：它不负责定义 Agent 的 personality，也不写复杂 workflow，而是告诉 Agent 当前机器、当前项目、当前 shell 下“哪些工具可用、怎么调用、哪里会不一样”。可以把它看作本地环境的 fact sheet。

## 问题

Agent 默认推理环境时，常见失败模式包括：

- 依赖通用路径：`/usr/bin/python3`、`~/.local/bin`、`C:\Python312` 在不同系统/发行版里差异很大。
- 分不清 shell 语法：PowerShell、bash、zsh、fish 的激活脚本和环境变量写法不同。
- 误用包管理器：同一项目可能同时存在 uv、poetry、pip、conda，Agent 随便选一个会破坏依赖。
- 找不到 MCP/插件入口：配置分散在 `~/.config/openclaw`、项目 `.mcp.json`、全局插件目录，工具名与实际命令不一致。
- 不感知平台差异：Windows 的路径分隔符、`venv` 激活脚本、`which` vs `where.exe`。

这些差异如果靠每次 prompt 临时描述，既啰嗦又容易漂移。tools.md 就是为了把“环境事实”固定下来。

## 做法/步骤

### 1. 选一个固定位置

建议区分两层：

- 全局：`~/.config/openclaw/tools.md`，记录机器级事实。
- 项目：`<repo>/tools.md` 或 `<repo>/.openclaw/tools.md`，记录项目级事实。

让 Agent 在会话开始或执行任务前读取这两个文件。如果项目文件存在，优先于全局；两者冲突时以项目文件为准。

### 2. 写最小可用内容

tools.md 不建议写太长，重点覆盖六类信息：

```markdown
# tools.md

## 运行环境
- OS: macOS 14.5 / Ubuntu 24.04
- Shell: zsh (默认), bash (脚本)
- 架构: arm64

## 解释器与包管理器
- Python: 使用 `uv run python`，不要直接调用系统 python3
- Node: 使用 fnm 管理，默认版本 20
- 项目依赖: 由 uv.lock 锁定，禁止直接 pip install

## 路径
- 项目虚拟环境: .venv
- 本地 bin: ~/.local/bin
- 缓存: ~/.cache/openclaw

## 环境变量
- OPENCLAW_HOME=~/.config/openclaw
- 不要覆盖: PATH、VIRTUAL_ENV

## MCP / 插件工具
- 文件操作: 用本地 shell 命令，不通过 MCP 包装
- 浏览器控制: mcp__browser__goto
- 代码检索: mcp__ripgrep__search

## 已知差异 / 失败处理
- Windows 上 venv 激活用 `.venv\Scripts\Activate.ps1`
- 如果 `uv run` 不存在，先执行 `curl -LsSf https://astral.sh/uv/install.sh | sh`
- 端口 3000 被占时不要杀进程，先询问用户
```

### 3. 让 Agent 真正读取

只写文件不够，需要把读取动作固化：

- 在 system prompt 中加入：`执行任何 shell/Python/Node 命令前，先读取 ~/.config/openclaw/tools.md 和当前项目 tools.md；如果没有找到，按默认环境执行并说明假设。`
- 如果 OpenClaw 支持 MCP 工具或插件，可以做一个 `read_tools_md` 的小工具，返回文件内容并附带最后修改时间，避免 Agent 忽略。
- 对自动化 pipeline，可以在任务开始时将 tools.md 内容注入上下文，减少一次工具调用。

### 4. 用脚本生成和校验

手工维护容易过时。建议加一个生成脚本，例如 `scripts/gen_tools_md.sh` 或 Python 脚本，采集 OS、shell、版本、路径、包管理器、MCP 配置，输出模板；人工只补“已知差异”部分。

同时可以在 CI 里加一步校验：如果 tools.md 中声明的命令在本地不存在，或关键路径不存在，就报 warning，提醒更新。

## 踩坑点

1. **硬编码用户名/主机名**：`/home/alice/project` 换机器就失效。尽量用 `~`、`$HOME`、环境变量表示。
2. **把密钥写进 tools.md**：API key、token、cookie 一律不写。Agent 需要时从环境变量或 secret store 读取。
3. **内容过长**：tools.md 超过 200 行后，Agent 可能只读一半或抓不住重点。拆成 base + machine-specific，保持每个文件短小。
4. **只写“应该怎样”，不写“实际怎样”**：例如写“使用 Node 20”，但机器实际是 Node 18，Agent 会困惑。脚本生成能减少这种漂移。
5. **忽略 Windows/PowerShell 差异**：Linux/macOS 用户写 tools.md 时容易只写 bash 语法。如果团队有 Windows 机器，需要明确 PowerShell 的等价命令。

## 可复用建议

- 把 tools.md 纳入版本控制，但全局文件可以不提交；项目文件必须提交。
- 用注释标出“事实”和“偏好”：事实是 OS/路径/版本，偏好是“优先用 uv 而不是 pip”。
- 定期重跑生成脚本，更新版本号与路径。
- 遇到一次新的环境错误，就把原因和解决办法追加到“已知差异”里，这是最高性价比的维护方式。
- 如果你的 Agent 支持多 machine profile，可以用 `tools.base.md` + `tools.<hostname>.md` 叠加，避免一份文件到处复制。

## 总结

tools.md 不是银弹，它管不了逻辑错误，也替代不了测试。但它能显著降低 Agent 在本地环境上“猜错”的概率。把环境差异写成结构化、可校验的事实，让 Agent 少踩路径、shell、包管理器和 MCP 入口的坑，是自动化实践里投入产出比很高的一件事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/44c0a928d601844c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/0c7213da2c8f25a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/6d40daa2df1151d2.png)

