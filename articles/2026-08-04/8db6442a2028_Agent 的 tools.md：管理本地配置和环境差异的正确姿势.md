---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31522
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

当 Agent 从一个项目目录部署到另一个环境，或者从开发机搬到云服务器时，最先崩掉的往往是工具定义。最初版本的 `tools.md` 通常是文字描述 + 代码块，本地跑通后直接推上去，结果远端 Agent 行为开始漂移：调用了不存在的路径、加载了错误的 Python 环境、MCP 配置指向了旧地址。

OpenClaw 的 Agent 依赖 `tools.md` 这份纯文本协议来裁剪和编排工具调用，但它自身不具备环境感知能力。工具描述里的任何硬编码，在换机器后都会成为潜在的语义毒药。

## 问题

`tools.md` 面临的典型环境差异包括：

1. **语言与包管理器路径**：本地有 `pyenv`，服务器用 `/usr/local/bin/python3`；本地 `brew` 安装的 Python 与系统内置 `python3` 混用。
2. **MCP / 外部服务端点**：本地 `localhost:8080` 的 MCP 服务，在云端是容器内网地址。
3. **项目根目录**：不同机器上的检查点、数据目录、日志目录位置完全不同。
4. **凭据与 API Key**：`.env` 文件位置不同，加载方式不同。
5. **操作系统语法差异**：`ls -G` 在 macOS 可用，Linux 上不识别；`sed -i` 在 macOS 需要 `''` 参数。

把这些写死在 `tools.md` 里，Agent 就会在所有机器上尝试同一套指令。跨环境失败不是模型问题，是配置文件欠缺抽象能力。

## 做法

### 1. 分层声明：静态描述 + 环境变量注入

`tools.md` 里将工具定义拆成两个区域：

- **常量区**：工具名称、用途、参数 schema、MCP 服务标识。
- **动态区**：所有与机器相关的路径、端口、包管理方式，统一使用 `${VAR}` 占位符。

```markdown
## 工具列表

### 数据流水线工具集
- 用途：ETL 与数据校验
- 工具：data_pipeline
- 协议：MCP
- 服务器定义：`${TOOLS_MCP_CONFIG}/servers.json`
- 执行环境：${PYTHON_EXECUTABLE}（激活 ${ENV_NAME} 虚拟环境）
- 可写目录：${DATA_DIR}/processed
```

### 2. 模板 + 条件渲染

`tools.md` 不直接作为最终产物，而是作为 Jinja2 模板。维护一个 `tools.md.j2`，部署时按机器类型渲染：

```jinja2
{% if platform == "macos" %}
- 依赖安装：brew install jq
{% elif platform == "linux" %}
- 依赖安装：apt install jq
{% endif %}
```

渲染后的 `tools.md` 是「当前机器专属」的静态快照。每次部署都重新渲染，避免给 Agent 喂入与代码库无关的信息。

### 3. 配置中心

OpenClaw 的配置读取需要支持外部注入。在项目的 `.openclaw/config.yaml` 或环境变量中指定：

```yaml
agent:
  tools:
    template: "config/tools.md.j2"
    vars:
      PYTHON_EXECUTABLE: "${PYTHON_EXECUTABLE}"
      DATA_DIR: "${DATA_DIR}"
      TOOLS_MCP_CONFIG: "${OPENCLAW_MCP_DIR}"
```

渲染发布后，需要确认 `tools.md` 里的变量已被替换，且容器路径可写。

```bash
python -c "import os; print(os.path.expandvars(open('tools.md').read()))" | grep -E "^\s*- (工具|服务器|执行环境|可写目录)"
```

### 4. 显式声明平台前置条件

在 `tools.md` 开头加一段「环境前置说明」，帮助 Agent 快速验证自身运行环境：

```markdown
## 环境前置（首次调用前必须确认）
- 当前平台为 {{ platform }}，Shell 解析方式为 {{ shell_style }}
- Python 解释器必须支持 type hints（要求 3.9+，路径见工具描述）
- 如 `${OPENCLAW_MCP_DIR}/servers.json` 不存在，先运行 ./scripts/setup_tools.sh
- 若 MCP 端点不可达，不要重试超过 3 次，回退到本地 CLI 实现
```

## 踩坑点

### 1. 把 API Key 写进 tools.md

`tools.md` 是给大模型看的，一旦 Key 出现在上下文中，就等于写进了日志、对话存档，而且模型在生成命令时可能将其拼入位置无关的调用，导致凭证膨胀。

**正确做法**：用 `${API_KEY}` 占位，实际密钥放在 `.env` 由启动脚本加载。

### 2. 盲写绝对路径

`/Users/yourname/projects/foo` 这种路径在服务器上没有生存空间。需要让 Agent 知道它是从哪个路径被调用的，在启动时把工作目录注入为变量。

```markdown
- 工作目录：${PROJECT_ROOT}
- 相对路径基准：${PROJECT_ROOT}/scripts
```

### 3. 环境差异不做条件分支

最常见的错误是只写变量，不告诉 Agent 何时该用哪种加载策略。需要显式声明：

```markdown
- 在 macos 上，Python 路径为 `${PYTHON_EXECUTABLE}`；在 Linux 服务器上，请使用 `.venv/bin/python`
```

### 4. 忽视 MCP 配置漂移

MCP 服务器配置和 tools 描述之间通常有同步关系。改了 `servers.json` 但没更新 `tools.md` 的服务器定义，Agent 就会用旧 schema 调新工具。

**建议**：让 MCP 配置文件的哈希进入 `tools.md` 的一行注释里，启动时校验：

```markdown
<!-- mcp-config-sha256: {{ mcp_config_hash }} -->
```

## 可复用建议

1. **强制模板化**：任何新工具必须先写成 `.j2`，再渲染成 `tools.md`，禁止直接把非模板内容提交到仓库。
2. **双环境验证**：在 CI 里跑一次 macOS + Linux 双环境渲染，任一环境渲染失败则阻断合并。
3. **可诊断性**：渲染后的 `tools.md` 末尾追加「未定义变量检查」段落，列出所有替换尚未完成的值，便于 Agent 自我诊断。
4. **环境信息自检工具**：以 Agent 工具形式暴露一个 `diagnose_env`，返回当前机器平台、变量解析结果、MCP 连通性测试。Agent 在启动后首先调用它，再开始处理任务。

这类做法可以用在每天的基础编辑工作流中，也能复制到 OpenClaw 之外的其他 Agent 项目。把 `tools.md` 当作代码来对待——写测试、加 CI、控制变更——跨环境的石头就会少很多。

---

