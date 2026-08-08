---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 32180
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：为什么环境差异是 Agent 落地工程化的最大绊脚石

在 OpenClaw、MCP 或自定义 Agent 开发中，几乎每个 Agent 最终都会调用本地工具：读取配置文件、执行 Shell 脚本、调用 Python 解析器、操作 Git。这些能力让 Agent 从“聊天机器人”变成真正能编码、能运维的自动化节点。

但问题也正出在这里：**Agent 所依赖的本地工具链在不同的开发机、CI 环境、生产主机上几乎不可能完全一致**。某个 Python 脚本在开发者的 Mac 上能跑，在 Docker 容器里却找不到正确的 Python 路径；Shell 命令依赖的 jq 在某个机器上版本过低，导致解析失败；甚至某些环境缺少必需的 CLI 工具，Agent 反复失败却只给出模糊的“命令未找到”。

这些差异无法通过单一的 `requirements.txt` 或 `Dockerfile` 完全消除，因为 Agent 的运行时环境往往已有既存状态。我们需要一种**声明式、机器可读、且能被 Agent 自身理解**的方式，来集中描述“我的环境里有什么工具、它们如何被调用、需要注意什么”。

## tools.md 方案：一份文件串联人与 Agent 的共识

**`tools.md` 是一份放在项目/Agent 工作目录下的 Markdown 文件，它描述当前环境可用的所有本地工具、它们的位置、版本要求、参数规则以及已知的跨环境差异。** 它既给人类工程师看，也给 Agent 的系统提示词或 MCP Server 读取。

一个典型的 tools.md 条目如下：

```markdown
## tool: python_runner
- executable: python3
- path_check: /usr/bin/python3, /opt/homebrew/bin/python3
- version_pattern: "Python 3\\.9\\+"
- description: Execute Python 3.9+ scripts. Always use `-u` for unbuffered output.
- env_vars: PYTHONIOENCODING=utf-8
- known_issues: |
  - On ARM macOS, the default python3 might point to Xcode's stub (check with `xcrun --show-sdk-path`).
  - In alpine containers, Python 3 is often named `python3`.
- examples:
  - `python3 -u script.py`
```

这个文件不追求面面俱到，但要求**覆盖 Agent 实际会调用的所有系统级命令和高频脚本**。

## 工程化做法：三步让 tools.md 真正可用

### 1. 定义标准 Schema 和存放位置

为了让 MCP Server 或启动脚本稳定地读取，tools.md 的结构需要可解析。推荐使用固定的 Markdown 章节标题 + YAML front-matter 的混合方式（GitHub、HuggingFace 上很多成熟项目都采用类似方法）。至少包含以下字段：

- `executable`：实际要执行的命令名（在 PATH 中或绝对路径）。
- `path_check`：多个候选路径，Agent 可据此检查工具是否存在。
- `version_command`：如何获取版本号（例如 `--version`），可选。
- `version_pattern`：一个正则或约束，用于检查版本是否满足。
- `env_vars`：需要设置的环境变量。
- `known_issues`：跨环境的常见陷阱，Agent 可以据此给出自愈建议。
- `examples`：至少一个正确使用示例。

存放位置建议：

- 单项目：`<project_root>/agent/tools.md` 或直接放在根目录。
- 多环境：在文件名中带上 profile，如 `tools.ci.md`，并在 Agent 启动时通过参数指定。

### 2. 在 Agent 启动流程中集成检查

不要期望 Agent 在每次工具调用时临时读取 tools.md（那样既慢又不可靠）。正确做法是**在 Agent 启动或 MCP Server 初始化时，将 tools.md 的内容解析为内部状态**。

以 MCP 为例，可以在 Server 的 `list_tools` 响应中动态生成工具列表，同时附加 `description` 中包含关键的环境提示。如果工具缺失，直接在 Server 启动阶段报错并给出修复命令，而不是等到 Agent 调用时才暴露异常。

一个简单的启动检查脚本（Python 伪代码）：

```python
import subprocess

def check_tool(path, version_cmd, version_pattern):
    # 1. check path existence
    # 2. run version_cmd
    # 3. match version_pattern
    # 如果失败，抛出明确的 MissingToolError 并引用 tools.md 中的 known_issues
```

这样**把“环境即代码”的检查左移到了 Agent 启动流程**，避免运行时反复踩坑。

### 3. 将 tools.md 注入 Agent 的 System Prompt

对于无法改造 MCP Server 的场景（例如直接用 OpenAI function calling 模式），可以将 tools.md 的内容以结构化的格式追加到 System Prompt 中，并明确指示：“如果你需要使用本地工具，请严格遵循 tools.md 中列出的命令和路径。”

这部分必须控制长度，否则会挤占 precious token 空间。可以把 `known_issues` 保留，而 `examples` 只在 Agent 初次调用失败时由另一个上下文检索机制加载。

## 踩坑记录

- **路径白名单不是银弹**：`path_check` 列表只能覆盖常见位置。遇到非标准安装时，Agent 仍可能找不到工具。更好的做法是让 Agent 具备“探测-反馈-重试”的能力：第一次失败后读 tools.md 的 `known_issues`，并尝试用 `command -v` 或 `which` 寻找。
- **环境变量作用域**：Agent 进程内设置的环境变量不一定能传递到它的子进程（取决于启动方式）。tools.md 中的 `env_vars` 要明确是子进程环境，并在实际执行时通过 `env=` 参数显式注入。
- **静态文件与实际环境的漂移**：tools.md 很容易被开发者遗忘更新。建议在 CI 中增加一个轻量级 Job，定期执行 `tools check`，并对比 tools.md 中声明的工具集合与实际 `PATH` 中的差异。出现漂移就告警。
- **Windows 路径转义与正斜杠/反斜杠**：如果你的 Agent 横跨 Linux/macOS/Windows，务必在 tools.md 中注明路径是 POSIX 风格还是 Windows 风格。MCP 的 transport 层通常统一处理，但 Shell Command Tool 仍需要手动适配。

## 可复用建议

1. **提供 `tools-check` 脚本**：一个独立的小脚本，读取 tools.md 并返回所有工具的可用性状态（JSON 格式）。Agent 可以将其作为第一个功能调用，甚至用于自我修复。
2. **分层覆盖**：`tools.base.md` 写通用工具，`tools.ci.md` 或 `tools.<hostname>.md` 写特定机器的覆盖。启动时合并，环境特异性覆盖默认值。
3. **纳入 MCP 注册表的工具包装**：将 tools.md 抽象为一种 MCP 工具类型，让它成为 Server 的“元工具”。当 Agent 想知道环境里有什么，直接调用 `list_local_tools` 即可。
4. **版本绑定**：tools.md 自身也需要版本号，建议纳入项目 Git 版本管理。当 Agent 报告环境问题时，可以连带 tools.md 的 commit hash 一并输出，便于回溯。

## 总结

`tools.md` 看似只是一个简单的 Markdown 文件，但它是 Agent 与真实世界环境之间最直接的“契约”。通过声明式描述本地工具、强制性启动检查、以及将契约注入 Agent 上下文，我们把难以控制的异构环境转换成了可控的、可修复的已知约束。对于 OpenClaw 这类正在快速从 demo 走向业务落地的 Agent 框架，这类工程化细节恰恰是决定能否在生产环境稳定运行的关键。

与其让 Agent 在烟雾缭绕的环境中反复碰壁，不如给它一份清晰的环境地图。

---

