---
title: Agent 的 tools.md：把“这台机器能干什么”写清楚
feedId: 34446
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、MCP 和插件体系里，Agent 的很多能力来自本地进程：Python 脚本、Node CLI、ffmpeg、Chrome、Docker，以及通过 MCP 暴露的 server。模型可以推理出命令，但很难推理出当前机器的真实环境。同一份任务在笔记本能跑，在 NAS 或 CI 容器里失败，往往不是因为 prompt 写得差，而是环境假设错了。

## 问题

常见失败模式：

- Agent 默认使用 ffmpeg，但机器没装；
- 调用 `python3` 实际只有 `python`，或版本从 3.11 升到 3.13 后依赖行为变化；
- 浏览器自动化时写死 `/usr/bin/google-chrome`，macOS 上路径完全不同；
- MCP server 配置在一台机器能用，另一台因为 `npx` 需要 `--yes` 或 Node 版本低而启动失败；
- 某些目录只读、不能 sudo、端口被占用等限制，Agent 不知道，反复尝试后失败。

这些差异如果散落在启动脚本、`.env`、插件配置、个人记忆里，排障成本很高。`tools.md` 的作用是把它们集中成一份面向 Agent 的事实文件。

## 做法 / 步骤

### 1. 分层维护

建立分层文件：全局 `~/.openclaw/tools.md` 描述机器通用能力；项目级 `tools.md` 描述项目相关工具和差异；会话级可以在 prompt 中追加临时限制，例如“本次不要用 Docker，因为 daemon 挂了”。

### 2. 内容按区块写

推荐结构：

- OS / Shell / 包管理器
- 运行时：Python、Node、Go、Rust 的命令和版本
- 外部 CLI：git、docker、ffmpeg、pandoc 等，是否可用、路径、注意事项
- 浏览器与 GUI：Chrome/Chromium 路径、是否支持 headless、有无 Xvfb
- MCP server：名称、启动命令、健康检查、常见失败原因
- 网络与代理：是否需要 proxy、哪些域名直连
- 已知限制：不能 sudo、GPU 不可用、端口占用、磁盘空间小

每项控制在 1-2 行，优先写“缺失”和“限制”，而不是罗列所有可用命令。例如：

```markdown
## CLI
- ffmpeg: 6.1.1, /opt/homebrew/bin/ffmpeg
- docker: 可用，但当前用户不在 docker 组，需 sudo
- chromium: 仅在 headless 模式可用

## Known limitations
- 不能 sudo
- 8080 端口被本地 dev server 占用
```

### 3. 自动生成 + 人工维护

环境版本信息用手写容易过时。建议用脚本生成自动部分，人工补充“限制和约定”。

```bash
{
  echo "# tools.md generated at $(date '+%F %T')"
  echo "## Runtime"
  python3 --version 2>/dev/null || echo "python3 missing"
  node --version 2>/dev/null || echo "node missing"
  echo "## CLI"
  for cmd in git docker ffmpeg pandoc; do
    if command -v "$cmd" >/dev/null 2>&1; then
      echo "$cmd: $(command -v "$cmd")"
    else
      echo "$cmd: missing"
    fi
  done
} > tools.md
```

然后人工追加 MCP 和限制部分，并在文件顶部写上“最后更新时间”和生成命令。

### 4. 让 Agent 按需读取

在系统提示或任务说明中加一句：“先查阅 `tools.md` 中与当前任务相关的章节，再调用工具。”不要默认全文注入，避免占用上下文。可以在文件顶部写“如何阅读本文件”，说明优先读 Known limitations，再按任务类型找章节。

### 5. 与工具定义解耦

`openclaw.json` / `mcp.json` 只写通用命令，环境差异放 `tools.md`。例如 MCP 统一用 `npx`，但 `tools.md` 说明“本机 npx 缓存易被清空，启动 MCP 时需要加 `--yes`”。

## 踩坑点

- **把密钥写进去**：`tools.md` 会被拼进上下文，可能发送到模型或日志。只写环境变量名，不写值。
- **信息过时**：机器升级后 `tools.md` 没更新，Agent 拿到错误路径比没有文件更糟。建议纳入初始化脚本或定期校验。
- **路径转义**：Windows 反斜杠、空格和盘符在 Markdown/JSON 中容易出问题。统一用正斜杠或 POSIX 风格，并注明 Shell。
- **与 MCP 配置不一致**：同一路径在两处写不同值，Agent 按 `tools.md` 调用，MCP server 内部仍用旧配置。最好由一个生成脚本同时产出两者。
- **写得太长**：大而全的系统信息会让 Agent 忽略关键限制。保持短，优先写“缺失”和“限制”。

## 可复用建议

- 维护一个 `tools.example.md` 模板，真实 `tools.md` 加入 `.gitignore`，或仅提交非敏感部分。
- 给 MCP server 记录健康检查命令和常见失败原因。例如浏览器 MCP 检查 Chrome 路径是否一致；文件系统 MCP 检查目标目录是否可写。
- 定期跑一个 validate 脚本：检查 `tools.md` 里声明的命令是否真的存在，路径是否可执行，版本是否匹配。不一致就告警。
- 对 macOS / Windows / Linux 分别维护独立文件，但用同一个模板，减少格式差异。

## 总结

`tools.md` 不是万能配置，也不能替代环境管理工具。它解决的是“Agent 无法感知环境差异”的问题。把机器能力写清楚、保持短和可验证、与 MCP/插件配置同步，本地自动化任务的失败率会明显下降，排障路径也更清晰。真正有价值的不是文件本身，而是让环境差异从“每次任务都要猜”变成“一次维护，按需查询”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/59856e697f6ff2e7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/246080213038cf87.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/32573d2cdda1233f.png)

