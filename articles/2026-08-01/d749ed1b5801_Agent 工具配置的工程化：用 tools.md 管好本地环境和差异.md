---
title: Agent 工具配置的工程化：用 tools.md 管好本地环境和差异
feedId: 31137
source: 综合讨论
publishedAt: 2026-08-01
---

# 背景：你的 Agent 为什么总在别人机器上跑不起来？

在做 Agent 开发时，我们经常需要让模型通过调用本地工具来完成实际任务：执行一段 Python 脚本、运行 `ffmpeg` 转换格式、读取某个目录下的文件。这些工具可能是系统命令，也可能是自己写的 shell 脚本或二进制程序。为了让框架（比如基于 MCP 协议的 agent 宿主）能够发现并调用它们，我们一般会用一个声明文件来描述工具的接口、命令路径和参数。

社区里已经出现了几种约定，例如 `tools.md`、`tools.yaml` 或 MCP server 配置块。其中 `tools.md` 因为可读性好、可以直接放在项目根目录下，被很多协作项目采用。它的形式类似：

```markdown
# tools.md
## file-counter
- description: Count lines of code in a directory
- command: python3
- args: ["scripts/loc.py", "{directory}"]
```

问题在于：这份文件隐含了大量对本地环境的假设——`python3` 在哪个路径？系统上是 `python3` 还是 `python`？`scripts/loc.py` 的相对路径基准是什么？当另一位协作者从 Git 拉取代码后，运行 agent 时大概率会收到 `command not found` 或者脚本执行失败的错误。

**根源**：工具声明把“逻辑调用契约”和“具体环境细节”耦合在了一起。

---

# 问题拆解：三种典型的环境差异

1. **可执行程序路径不同**  
   macOS 上 `python3` 可能在 `/usr/local/bin`，Linux 上可能是 `/usr/bin/python3`，Windows 上是 `python.exe`。直接写死 `command: python3` 依赖 PATH，但如果 PATH 没配好，或者需要特定版本（如 `python3.11`），就会失败。

2. **脚本/资源文件位置依赖于项目根目录**  
   使用相对路径 `scripts/loc.py` 是常见做法，但当 agent 的工作目录（cwd）不是项目根目录，或者工具定义文件被符号链接指向时，相对路径解析会出错。

3. **跨操作系统差异**  
   路径分隔符、换行符、文件权限（例如 `.sh` 脚本在 Windows 下是否可执行）都会导致同样的 `tools.md` 产生不同结果。

这些问题在团队协作和 CI/CD 部署中会被放大。每换一个环境就要手动修改配置，不仅低效而且容易引入错误。

---

# 做法：让 tools.md 成为“环境无关的契约”

我们要做的是把环境相关的细节抽离出来，让 `tools.md` 只描述**工具的接口和业务语义**，而将具体命令路径交给运行时解析。

## 1. 引入占位符和默认值

将 `tools.md` 中的命令写成变量引用形式，并给出默认值：

```markdown
## code-stats
- description: Analyze code complexity
- command: ${PYTHON:=python3}
- args: ["${SCRIPTS_HOME:=./scripts}/complexity.py", "--dir", "{target}"]
```

解释：
- `${PYTHON:=python3}`：优先取环境变量 `PYTHON`，若未设定则使用 `python3`。
- `${SCRIPTS_HOME:=./scripts}`：同理，允许外部覆盖脚本根目录。

## 2. 提供环境配置模板

在项目根目录放置 `.env.example`：

```bash
# 指定 Python 解释器路径
PYTHON=/usr/local/bin/python3.12
# 脚本根目录（相对于项目或者绝对路径）
SCRIPTS_HOME=./scripts
# 如果是 Windows 下的特殊工具，也可以写：
# FFMPEG=C:\tools\ffmpeg.exe
```

协作者 clone 后，只需 `cp .env.example .env` 并根据自己机器调整路径，然后 agent 加载工具时读取 `.env` 文件替换变量即可。

## 3. 运行时解析

可以在 agent 启动前加一个轻量级的加载器（比如一个 `parse_tools_md.py`），它的职责是：
- 读入 `tools.md`，用正则匹配 `${VAR:=default}`。
- 从环境变量（或 `.env`）中获取实际值，若不存在就使用 `default`。
- 生成一份解析后的工具描述对象给 agent 框架使用。

这个加载器也可以用 shell 脚本实现，但 Python 代码能更方便地处理跨平台路径和文件权限检查。

## 4. 加入自检命令

为避免运行时才发现工具不可用，可以在 `tools.md` 中为每个工具增加一个 `check` 字段：

```markdown
## file-counter
- description: ...
- command: python3
- args: ["Scripts/loc.py", "{directory}"]
- check: python3 Scripts/loc.py --help
```

启动时运行所有工具的 `check` 命令，如果退出码非零，立即报告并阻止 agent 启动，而不是让它在任务执行中途报错。这对调试新环境非常友好。

---

# 踩坑记录

- **Windows 路径中的反斜杠**  
  在 `tools.md` 中写 `C:\tools\something.exe` 会被 Markdown 或 JSON 解析器转义。建议统一使用正斜杠，加载器在 Windows 上做转换，或者要求环境变量中配置的路径使用正斜杠或原始字符串。

- **环境变量的作用域**  
  如果 agent 运行在 Docker 容器或 systemd 服务中，`PATH` 和自定义环境变量可能与开发 shell 不同。需要明确把 `.env` 的加载逻辑放在最前面，并且注意不要依赖交互式 shell 才有的配置（如 `.bashrc`）。

- **命令本身的输出格式差异**  
  某些工具在不同版本中输出格式不同（如 `ffmpeg` 的 progress 信息）。这已经超出路径管理的范围，但提醒我们需要在工具描述中约定版本，并在自检时检测版本号。

- **相对路径的基准点**  
  我们的加载器应该保证所有相对路径都基于 `tools.md` 所在的目录解析，而不是当前工作目录。在实现时用 `os.path.dirname(__file__)` 或配置 `PROJECT_ROOT` 环境变量来固定基准。

---

# 可复用建议

1. **一项目一份 `tools.md` + `.env.example`**  
   将工具配置和默认环境变量模板纳入版本控制，环境差异归 `.env`（`.gitignore` 忽略）。

2. **提供一个“环境检测”脚本**  
   `python tools/check_env.py`：扫描 `tools.md` 中所有声明，检查命令是否存在、脚本文件是否可访问、权限是否正确。输出清晰的报告，帮助新人快速修复环境问题。

3. **在 CI 中强制校验**  
   CI 工作流里也运行 `check_env.py`，使用一个最小化的干净容器验证 `tools.md` 的默认值是否能正常工作，避免提交了只有本地才能跑的工具声明。

4. **考虑生成工具清单**  
   为团队提供一个简单的 Markdown 表格生成器，自动从 `tools.md` 拉取描述和前置依赖，方便协作者在部署文档中查阅系统要求。

---

# 总结

Agent 的工具配置管理看似只是一个小小文件，但它处于“模型智能”与“本地执行”的交界处，是环境差异最先爆发的地方。通过将 `tools.md` 变成环境无关的声明，引入变量替换、自检和能力检测，我们能够把协作者的“搭环境时间”从小时缩短到分钟级，也让 agent 从“在开发者机器上跑得通”真正迈向可移植交付。

记住这句话：**工具声明应是契约，不是环境的影子。** 下次你写 tools.md 时，先问自己一句：如果换了台机器，它还能活下来吗？

---

