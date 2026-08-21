---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 34087
source: 综合讨论
publishedAt: 2026-08-22
---

在 OpenClaw/Agent 实践里，经常遇到一个尴尬：同一个任务在 A 机器能跑通，换到 B 机器就路径不对、命令不存在、MCP 连接失败。原因往往不是 Agent 能力不够，而是本地配置没有被显式管理。`tools.md` 可以作为 Agent 读取的“本机能力与差异声明”，把环境信息从 prompt 和脚本里抽离出来。

## 背景

Agent 默认倾向于使用通用知识或训练数据里的路径，而不是你的实际环境。如果每次都把“项目在 `/Users/xxx/Projects/foo`”“Python 在 `/opt/anaconda3/bin/python`”写进 prompt，既容易泄露本机细节，又无法跨环境复用。`tools.md` 的作用是让 Agent 在执行前有据可查：这台机器有什么工具、路径在哪里、服务怎么连。

## 问题

本地配置差异通常来自三类：

1. **路径差异**：不同机器上项目目录、虚拟环境、数据目录位置不同。
2. **命令/版本差异**：macOS 有 `brew`，Linux 有 `apt`；同一命令的版本也可能不同。
3. **服务与凭据差异**：MCP server 的启动参数、本地 API 端口、健康检查地址不一致。

如果这些散落在 prompt、脚本或多个 yaml/json 里，排查成本会很高；如果直接把密钥写进工具描述，风险更大。

## 做法/步骤

建立一个 `tools.md`，放在 Agent 工作目录或项目根目录。内容分四段：

- **环境概况**：OS、shell、包管理器、CPU/内存等少量信息。
- **工具清单**：哪些 CLI 可用、版本、常用子命令。例如 `python3`、`ffmpeg`、`docker`。
- **本地路径**：项目目录、数据目录、缓存目录、虚拟环境路径，尽量使用变量而非绝对路径。
- **服务与 MCP**：如何启动/连接本地服务、MCP server 命令、健康检查方式。

一个最小模板：

```markdown
# tools.md

## Environment
- OS: linux
- Shell: bash
- Package manager: apt

## Local paths
- PROJECT_ROOT: /workspace/foo
- DATA_DIR: ${PROJECT_ROOT}/data
- PYTHON_ENV: ${PROJECT_ROOT}/.venv

## Tools
- python3: 3.12
- ffmpeg: 6.1
- docker: available

## Services / MCP
- playwright MCP: npx @playwright/mcp@latest --headless
- local API: http://127.0.0.1:8787/health
```

加载方式上，不建议把整个 `tools.md` 永久塞进 system prompt。更好的做法是：任务开始时让 Agent 读取一次；或者由入口脚本生成摘要后注入上下文。同时可以拆成两层：

- `tools.md`：静态能力，纳入版本管理。
- `tools.local.md`：机器专属配置，加入 `.gitignore`，不入库。

## 踩坑点

1. **密钥不要进 `tools.md`**。用环境变量引用，例如 `GITHUB_TOKEN=${GITHUB_TOKEN}`，不要在文件里放真实值。
2. **路径写法要统一**。Windows 的反斜杠和盘符容易被 Agent 误判，建议统一使用正斜杠或环境变量表达。
3. **文件会漂移**。新装工具或升级版本后，`tools.md` 很容易过期。可以用 `command -v python3`、`ffmpeg -version` 等命令做成校验脚本，在 Agent 启动前提示“`tools.md` 可能过旧”。
4. **不要写得过细**。每个工具都贴完整 `--help` 会浪费上下文。只写与当前工作流相关的命令和差异。
5. **注意中英文标点与换行**。`tools.md` 经常被当作代码块读取，混入中文标点或异常换行可能影响 Agent 解析。

## 可复用建议

- 采用“差异优先”写法：默认假设标准 Linux/macOS 工具可用，只写差异、非默认版本、专用脚本。
- 为每个服务写一行健康检查命令，例如 `curl -fsS http://127.0.0.1:8787/health`，方便 Agent 自检。
- 团队协作时，约定新机器初始化先用脚本生成 `tools.local.md`，而不是手动复制。
- 将 `tools.md` 作为项目文档的一部分，和代码一起评审、更新。

## 总结

`tools.md` 不是给 Agent 的“说明书大全”，而是本机配置的单一事实来源。它解决的是可复现性问题：让 Agent 在不同机器上先看清环境，再动手执行。管理好它，能显著减少路径错误、命令不存在的低级失败，也能避免密钥和本机细节泄漏到 prompt 里。对 OpenClaw/MCP/自动化实践来说，这比堆更多插件更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/8d42cc3354122766.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/25841e9f634904ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9d4041388bd35959.png)

