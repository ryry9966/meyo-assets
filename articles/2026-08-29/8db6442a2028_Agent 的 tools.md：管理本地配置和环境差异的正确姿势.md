---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35195
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 工作流里，模型负责规划与调用工具，但真正执行命令的是本地环境。同一个任务，在 A 机器能跑通，在 B 机器可能因为 `ffmpeg` 未安装、Python 路径不同、Windows 与 Unix 命令差异而失败。Agent 如果只能靠猜，容易生成不存在的路径或错误参数。`tools.md` 的作用就是把这些“本地事实”用可读、可版本化的方式固定下来，降低跨环境执行的不确定性。

## 问题：环境差异如何影响 Agent

常见失败模式有三类：

1. 路径硬编码：`/Users/name/...` 或 `C:\Users\name\...` 换机器就失效。  
2. 命令名与版本差异：`python` 和 `python3`、`pip` 和 `pip3`、`ffmpeg` 版本不一致导致参数不兼容。  
3. 环境变量缺失：API key、代理地址、数据目录未声明，Agent 执行到一半才发现缺配置。

这些问题的根源不是模型不够聪明，而是项目里缺少一份面向 Agent 的环境契约。

## 正确做法：把 tools.md 当成工具说明书

### 1. 先盘点，不急着写

列出 Agent 会接触的本地能力：CLI 工具、解释器、包管理器、MCP server、插件命令、常用脚本。对每一项记录四要素：

- 命令名称
- 最低版本或推荐版本
- 安装/获取方式
- 依赖的环境变量或路径

例如：

```markdown
## 工具清单
- ffmpeg
  - 命令: ffmpeg
  - 最低版本: 6.0
  - 安装: brew install ffmpeg / winget install ffmpeg
  - 验证: ffmpeg -version | head -n 1
```

### 2. 用环境变量和平台分支代替硬编码

不要在 tools.md 里写死绝对路径。约定项目内使用 `$PROJECT_ROOT`、`$DATA_DIR` 等变量，并在 `.env.example` 中给出默认值。对 Windows 和 Unix 分开描述：

```markdown
## 路径约定
- 项目根目录: $PROJECT_ROOT
- 数据目录: $PROJECT_ROOT/data
- Unix shell: bash/zsh
- Windows shell: PowerShell 7+
```

如果某个命令在两个平台不同，就写两个条目，例如 `python` 与 `py -3`。

### 3. 每个工具都要有预检命令

这是 tools.md 最有价值的部分。Agent 在执行任务前，可以先跑预检命令确认环境是否满足。建议在文件开头放一段“环境自检”：

```markdown
## 预检
- python --version
- node --version
- ffmpeg -version
- git --version
```

这些命令输出直接决定 Agent 是否继续，或是否需要提示用户安装。

### 4. 与 MCP/插件联动

如果项目使用了 MCP server 或本地插件，把 tools.md 的路径或入口写进配置说明。可以在 Agent 的 system prompt 或项目规则中要求“任务开始前先读取 tools.md”。例如在 OpenClaw 项目配置里加入：

```text
before_task: read tools.md and run preflight checks
```

这样 tools.md 就从静态文档变成了执行流程的一环。

## 踩坑点

- **不要把 secrets 写进 tools.md**：token、password、API key 应该引用 `.env`，tools.md 只记录变量名和用途。
- **文档与真实环境不同步**：环境升级后没人更新 tools.md，预检命令就会失效。建议配合 `preflight.sh` 或 CI 任务定期校验工具版本。
- **过度抽象**：不要试图写一份覆盖所有操作系统的“万能文档”。只记录项目实际使用的平台差异，未验证的内容不要写。
- **Windows 路径与空格**：在 YAML/JSON 配置中，`C:\Program Files\...` 需要转义或加引号，否则 Agent 拼接命令时会被空格截断。
- **版本约束太松**：只写“需要 Python”没有意义。写清最低版本，Agent 才能判断是否满足。

## 可复用建议

1. 团队维护一份基础 `tools.md` 模板，项目只覆盖差异部分。  
2. 把 tools.md 和 `.env.example` 一起纳入版本控制，保证新成员克隆后能快速对齐环境。  
3. 写一个 `scripts/check_env.sh` 自动运行预检命令，并输出机器可读结果，让 Agent 或 CI 调用。  
4. 环境变更时，先更新 tools.md 再提交代码，避免文档滞后。  
5. 让 Agent 在规划阶段就读取 tools.md，而不是执行到一半才去查。

## 总结

`tools.md` 不是配置中心，也不是替代 `.env` 或包管理器的方案。它是 Agent 与本地环境之间的“契约层”：用可验证的命令、平台分支和预检逻辑，减少模型对环境的猜测。把它写成简短的、可执行的事实清单，比写一大段说明更有效。对 OpenClaw/Agent/MCP 用户来说，这通常是从“能跑”到“稳定复现”的关键一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/eeff0edada45cdc1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/4bd0adccd48f2463.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/728ab22402e197c2.png)

