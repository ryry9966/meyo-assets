---
title: Agent 的 tools.md：本地配置与环境差异管理实践
feedId: 33700
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw、MCP 或插件型 Agent 的日常使用中，工具调用经常要触碰本地环境：读写项目文件、执行脚本、启动 MCP server、加载本地插件。同一个 agent 配置经常在一台机器上正常，换到另一台机器就失败。原因通常不是 prompt 写得不好，而是本地路径、环境变量、解释器位置、操作系统差异被硬编码进了工具描述或启动参数里。

`tools.md` 适合作为工具配置的“收口层”，把机器相关的内容集中管理，避免散落在 system prompt、脚本、tool schema 和 MCP 启动命令里。

## 常见问题

典型症状包括：

- Agent 使用 `/Users/alice/Code/...`，但同事机器上是 `/home/bob/repo`。
- 命令写死 `python3`，实际依赖装在某个 venv 里。
- MCP server 通过 `npx` 每次启动，拉包慢且版本漂移。
- 本地开发能跑，CI 或容器里因为缺少某个环境变量直接失败。
- Windows 路径反斜杠在 Markdown 或 JSON 里被转义，导致命令解析错误。

这些问题的本质是：工具配置没有区分“通用能力”和“本机实现”。

## 做法/步骤

1. **建立结构化 tools.md**

   为每个本地工具记录一组字段，例如：

   ```markdown
   ## tool: repo_reader
   - command: ${REPO_READER_BIN:-python3} ${OPENCLAW_REPO_DIR}/tools/read_repo.py
   - env: OPENCLAW_REPO_DIR, REPO_READER_BIN
   - platform: unix/windows
   - health_check: ${REPO_READER_BIN:-python3} -c "import yaml"
   ```

   关键点：只写变量表达式和默认值，不写绝对路径。

2. **用环境变量代替硬编码**

   所有机器相关内容使用 `${VAR}` 或 `${VAR:-default}` 形式：

   - `${HOME}`、`${USERPROFILE}` 表达用户目录。
   - `${OPENCLAW_WORKSPACE}` 表达当前项目根目录。
   - `${PYTHON_BIN:-python3}` 让机器可以覆盖解释器位置。
   - `${MCP_FILESYSTEM_ROOT}` 表达 MCP file server 允许访问的根目录。

3. **区分平台命令**

   如果必须支持 Windows 和 Unix，可以同时声明两套命令：

   ```markdown
   command_unix: ${PYTHON_BIN:-python3} ${TOOL_SCRIPT_PATH}
   command_windows: ${PYTHON_BIN_WIN:-python} ${TOOL_SCRIPT_PATH}
   ```

   优先通过 wrapper 脚本统一，避免在 tools.md 里写复杂 shell 语法。

4. **提供健康检查**

   每个工具最好附带 `health_check`。加载配置时，Agent 可以执行一次检查，生成状态表，例如：

   - `python -c "import httpx"` 检查依赖是否安装。
   - `ffmpeg -version` 检查二进制是否可用。
   - `npx --yes @modelcontextprotocol/server-filesystem --version` 检查 MCP server 是否可拉取。

   建议在 OpenClaw 侧做一个 `tools doctor` 子命令，遍历 tools.md 并输出 ready / missing / error。

5. **分层与本地覆盖**

   建议维护多个文件，合并优先级从低到高：

   - `tools.example.md`：入库模板，包含所有变量说明和健康检查。
   - `tools.project.md`：项目级共享配置，入库。
   - `tools.local.md`：个人本机配置，加入 `.gitignore`。

   真正的 `tools.md` 可以由这些文件合并生成，或者由加载器按顺序读取。

## 踩坑点

- **环境变量为空**

  `${API_BASE}` 为空时可能导致请求打到根路径或构造出错误 URL。建议始终加默认值或校验逻辑，例如 `${API_BASE:-http://127.0.0.1:8000}`。

- **Windows 路径正反斜杠**

  Windows 下 `C:\Users\alice` 在 Markdown 或 JSON 中容易被转义。建议统一使用正斜杠 `C:/Users/alice`，让命令由 shell 解释；或者在 wrapper 中处理路径转换。

- **Secret 泄漏**

  不要把 API key、token、数据库密码写进 tools.md。即使是本地文件，也可能被打包进日志、错误信息或分享截图。secret 只通过环境变量注入，tools.md 中只保留变量名。

- **venv 路径不一致**

  `python` 可能指向系统解释器，但依赖装在 `.venv` 里。tools.md 应记录 `VENV_BIN` 或直接使用 `.venv/bin/python` 的变量形式，避免运行时 import 失败。

- **MCP server 启动方式漂移**

  `npx` 每次启动可能拉取最新版本，导致行为变化。建议在 tools.md 中固定版本号，或记录本地安装路径，例如使用 `bunx --bun` 或全局安装后的命令。

- **健康检查误报**

  部分工具旧版本 `--version` 返回非 0 退出码，即使工具可用。健康检查不要只看退出码，应同时检查 stdout/stderr 内容。

## 可复用建议

- 把 tools.md 当作**接口**，而不是说明文档：字段结构化，便于解析和 diff。
- 工具描述尽量简短，环境差异集中在 tools.md，不要塞进 system prompt。
- 在 CI 中运行一次 `tools doctor`，失败则阻断，避免运行期才发现环境问题。
- 每次修改 tools.md 后，用 `diff` 检查是否出现绝对路径、真实 secret 或多余平台分支。
- 在容器镜像构建时生成默认的 `tools.local.md`，减少运行时探测成本。

## 总结

tools.md 的价值不在于罗列工具，而在于把环境差异压缩到一个可审查、可覆盖、可校验的配置文件里。先抽象变量，再写平台分支，最后用健康检查兜底。这样做不能消灭所有“我机器上能跑”的问题，但能把大部分本地配置问题从运行时移回加载期，让 Agent 工具链在不同环境下更可预期。

---

