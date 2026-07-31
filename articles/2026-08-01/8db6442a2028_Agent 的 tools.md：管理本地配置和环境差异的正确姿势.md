---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31133
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：当 Agent 走进本地环境

给 Agent 接入本地工具非常诱人——一条指令就能操作数据库、调用 CLI、读写文件，自动化场景瞬间打开。但在实际工程里，不同机器之间的环境差异很快会成为“执行失败”的最大来源。Python 版本、系统包管理器、路径分隔符、动态库缺失、代理设置……这些只有在调用那一刻才会暴露。

如果你已经用过 OpenClaw 或 MCP 插件，大概率遇到过类似情况：开发机 Ubuntu 一切正常，换到同事的 macOS 就报 `command not found`；CI 的 Node 版本低了两个大版本，Agent 直接罢工；或者同一个工具在 Windows 下需要 `where` 而非 `which`。

本质上，Agent 的可移植性受制于我们对本地依赖的描述方式。多数工具声明只给了命令名称，环境检测和修复靠运气。因此我们需要一种更好的配置实践，让 Agent 能在调用工具前，主动适应本地环境。

## 问题：硬编码假设与沉默失败

常见的工具配置（无论 MCP server 字段还是 OpenClaw 的 tool 定义）往往只关心命令是否存在，忽略更细粒度的环境约束：

- **版本假设**：`python` 可能指向 2.7，但脚本需要 3.9+。
- **路径差异**：`docker` 在 Linux 是 `/usr/bin/docker`，在 macOS 可能是 `/usr/local/bin/docker`，而 Agent 只检查最弱的一种。
- **系统工具缺失**：依赖 `jq`、`curl`、`make`，但某些精简发行版并未预装。
- **运行时依赖不透明**：Python 包、Node 全局模块、系统共享库——它们在工具脚本里隐形存在，调用失败时 Agent 只会收到一个退出码，调试信息极度匮乏。

所有这些导致的是“沉默失败”或误导性错误，修复成本远高于预防成本。我们需要一种声明式、可执行的环境契约，让 Agent 在初始化或每次调用前完成自检与自适应。

## 做法：用 tools.md 建立环境契约

这里的 `tools.md` 是一个工程文件（实际可以是 YAML/TOML，命名为 `tools.md` 便于 Agent 解析 Markdown 代码块），用来描述每一个本地工具所需的运行环境、检测方法、修复指令以及回退策略。

### 1. 定义工具和环境要求

以 YAML 块嵌入 Markdown 为例：

```yaml
tools:
  - name: python-script-runner
    type: local-cli
    command: python3
    required_version: ">=3.9,<3.13"
    detect:
      linux: ["python3", "--version"]
      darwin: ["python3", "--version"]
      win32: ["python", "--version"]
    install:
      linux: "sudo apt install -y python3 python3-venv"
      darwin: "brew install python@3.12"
      win32: "winget install Python.Python.3.12"
    pre_check: "pip list | grep requests || pip install requests"
    fallback: "use-container python:3.12-slim"
```

这个定义把环境需求从代码里抽离出来，成为可维护的文档，同时 Agent 可以解析并执行检测逻辑。

### 2. 在 Agent 启动时执行环境自检

OpenClaw 可以通过一个系统工具（如 `env-checker`）在启动或工具注册阶段读取 `tools.md`，依次执行 `detect` 命令，解析版本号，若不符合则根据 `install` 提示用户或自动安装。实现上可以分两层：

- **轻量检测**：只验证命令存在且版本匹配，避免每次调用都全量检测。
- **深度预热**：工具第一次调用前执行 `pre_check`，确保运行时依赖就绪。

### 3. 平台感知与 fallback 机制

利用操作系统标识（`GOOS` 或 `platform.system()`）选择对应的 `detect`/`install` 块。同时为每个工具配备 fallback，例如 Docker 容器执行命令，这样在本地环境无法快速修复时，Agent 仍能完成工作，而不是直接失败。

### 4. 把 tools.md 纳入版本控制

`tools.md` 应作为仓库的一部分，随代码一起演进。任何人 clone 项目后，只需运行 Agent 就能得到一致的工具环境提示，而不是靠口口相传的“你先装一下 xxx”。

## 踩坑点

1. **版本字符串解析不规范**  
   `python3 --version` 的输出在不同系统可能包含额外文字，如 `Python 3.11.2`，解析时务必用正则提取数字，再做语义化比较。避免 `3.10` < `3.9` 的字符串比较陷阱。

2. **安装命令权限与交互**  
   自动执行 `apt install` 或 `brew install` 可能因权限不足卡住，或弹出交互式确认。Agent 必须使用 `-y`/`--no-interactive` 并明确提示用户手动执行，避免静默失败。

3. **Windows 路径和引号**  
   在 `detect` 数组里写入 shell 命令时，Windows 的 cmd 和 PowerShell 对引号、路径空格的处理不同。建议明确标注 shell 类型，或直接使用跨平台脚本语言（Python）实现检测。

4. **网络依赖**  
   `pre_check` 中 `pip install` 可能因网络或代理失败。tools.md 应允许配置代理环境变量，或通过注释提示离线环境下的准备步骤。

## 可复用建议

- **与 OpenClaw 工具注册结合**：在注册工具时，读取对应的 tools.md 条目，仅当环境检测通过才暴露工具给 Agent，避免无效调用。
- **生成报告而非静默修复**：让 Agent 输出环境差异报告，由用户决定是否自动修复，保持可控性。
- **工具状态缓存**：首次检测后将结果写入本地缓存文件（`.agent_env.json`），后续启动只需哈希比对 tools.md 是否变更，避免反复检测。
- **贡献给团队**：将 tools.md 模板和检测脚本封装成 MCP 插件 `agent-env-doctor`，其他项目可直接引用，减少重复劳动。

## 总结

管理本地配置和环境差异不是新问题，但放到 Agent 调用链里，影响会被放大——因为你不再直接面对终端输出，而是面对 Agent 的“我不知道错在哪”。`tools.md` 提供了一种工程化的声明方式，把谁、在哪、靠什么运行这个事说清楚，让 Agent 的本地工具调用从碰运气变成可预期的行为。

这份文件很小，但它可能是让你的自动化脚本从“能跑”变成“到处能跑”的关键一环。

---

