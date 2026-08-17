---
title: Agent 的 tools.md：把本地环境差异写成一份可执行的说明书
feedId: 33631
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw/Agent 的日常实践里，工具配置越来越分散：MCP server 写在 `mcp.json`，脚本路径写在 prompt 里，环境变量丢进 `.env`，本机可用的 CLI 工具又经常依赖 shell profile。结果就是同一套 Agent 配置，在 A 机器上能跑，换到 B 机器就开始“工具不存在”“路径找不到”“MCP 启动失败”。

这类问题通常不是 Agent 能力不够，而是 Agent 缺少一份对“当前机器环境”的描述。tools.md 要解决的就是这件事：它不是工具注册表，而是本地配置与环境差异的适配层。

## 问题

常见的翻车现场包括：

- macOS 上用 `brew` 装的 `ffmpeg` 在 Linux 机器上路径不同；
- 本地 MCP server 依赖 Node 18，但目标机器还是 Node 16；
- Windows 下路径反斜杠、空格、盘符没有处理；
- `.env` 里只写了 key 名，没写当前环境该用什么值；
- Agent 不知道某个脚本需要先激活虚拟环境或启动本地服务。

如果这些信息只存在人的脑子里，或者散落在聊天记录里，Agent 每次执行任务都会踩坑。

## 做法：把 tools.md 当作环境说明书

### 1. 结构不要大而全，只写“差异”

一份可用的 tools.md 建议包含四块：

```markdown
# tools.md

## 运行环境
- OS: darwin / linux
- shell: zsh
- node: 20.11.0
- python: 3.12.2

## 工具清单
| 工具 | 用途 | 本地命令/路径 | 环境要求 | 验证命令 |
| --- | --- | --- | --- | --- |
| ffmpeg | 音视频处理 | /opt/homebrew/bin/ffmpeg | PATH 已注入 | ffmpeg -version |
| browser-mcp | 浏览器自动化 | npx @openclaw/browser-mcp | node >=18 | npx @openclaw/browser-mcp --version |
| local-api | 内部接口 | http://127.0.0.1:8787 | 需先启动脚本 | curl -s http://127.0.0.1:8787/health |

## 本地启动命令
- 先执行: source .venv/bin/activate
- 再执行: ./scripts/start-local-stack.sh

## 已知差异
- Linux 服务器上没有 pbcopy，剪贴板操作统一走 xclip。
- 本机默认 shell 是 zsh，远程执行时显式使用 /bin/bash。
```

关键原则：不写“这是什么工具”的通用介绍，只写“这台机器上这个工具怎么用、和默认有什么不同”。

### 2. 让 Agent 在合适的时机读取

不建议把整个 tools.md 永远塞进 system prompt，尤其是工具很多的项目。更好的方式是：

- 把 `tools.md` 放在项目根目录，或 `~/.config/openclaw/tools.md`；
- 在任务指令里要求 Agent 开始调用本地工具前先读取相关段落；
- 如果 Agent 支持文件检索，让它按工具名或任务类型抓取对应片段。

例如任务 prompt 可以写：

> 执行前先阅读项目根目录 tools.md，确认本机可用命令、路径和环境变量。不要根据默认假设调用工具。

这样比把环境信息硬编码进每条 prompt 更容易维护。

### 3. 用脚本生成和校验

tools.md 最怕失真。推荐准备一个简单的生成脚本，把本机真实状态写入文件：

```bash
#!/usr/bin/env bash
{
  echo "## 运行环境"
  echo "- OS: $(uname -s)"
  echo "- shell: $SHELL"
  echo "- node: $(node -v 2>/dev/null || echo missing)"
  echo "- python: $(python3 --version 2>/dev/null || echo missing)"
  echo ""
  echo "## 工具状态"
  echo "| 工具 | 状态 | 验证命令 |"
  echo "| --- | --- | --- |"
  for cmd in ffmpeg docker npx curl; do
    if command -v $cmd >/dev/null 2>&1; then
      echo "| $cmd | ok | $($cmd --version 2>/dev/null | head -n 1) |"
    else
      echo "| $cmd | missing | - |"
    fi
  done
} > tools.md
```

生成后提交到版本库，换机器时重新跑一遍。这样 Agent 读到的就是接近真实的环境信息。

## 踩坑点

1. **把 secrets 写进 tools.md**  
   API key、token、内网密码不要出现。tools.md 经常被整段注入上下文，泄露风险很高。敏感信息继续走环境变量或 secret 管理。

2. **路径写死且没有 fallback**  
   `/opt/homebrew/bin/ffmpeg` 在 Linux 上不存在。可以写成“优先用 `which ffmpeg`，找不到再试 `/opt/homebrew/bin/ffmpeg`”，让 Agent 有探测能力。

3. **只有文档，没有验证**  
   tools.md 里写“已验证”，但实际命令已经失效。最好让 Agent 在执行关键任务前先跑一遍 validation command，失败就停止并报告，而不是盲目继续。

4. **一大篇全部注入上下文**  
   如果项目有 20 个工具，整份 tools.md 会占掉大量 token，也容易干扰推理。按需加载是更工程化的做法。

5. **和 MCP 配置重复且不一致**  
   `mcp.json` 里已经定义了 server 启动方式，tools.md 又写一套，时间长了必然分叉。tools.md 更适合记录“本机差异”和“验证方式”，不要复制完整 MCP 配置。

## 可复用建议

- **分层管理**：全局 `~/.config/openclaw/tools.md` 写机器级信息，项目内 `tools.md` 写项目级差异。
- **状态字段**：给每个工具加 `ok / missing / untested` 状态，Agent 遇到 `missing` 时优先提示人工处理。
- **动态生成**：不要手写路径，用 `which`、`command -v` 的结果生成，减少路径漂移。
- **团队评审**：tools.md 的修改和代码一起走 PR，环境变更可见、可追溯。
- **小步验证**：新机器第一次跑 Agent 前，先让 Agent 输出一份“环境检查摘要”，确认工具状态后再执行正式任务。

## 总结

tools.md 的核心价值不是“告诉 Agent 有哪些工具”，而是“告诉 Agent 这台机器和标准假设有什么不同”。它能显著减少跨机器、跨开发者之间的工具失效问题，代价只是一份轻量文档和一个生成脚本。

工程上更推荐的做法是：tools.md 尽量由脚本生成，只描述差异，按需注入，并且永远不写入敏感信息。这样 Agent 在不同环境下会更稳，维护成本也更低。

---

