---
title: Agent 的 tools.md：把本地环境差异关进笼子
feedId: 34982
source: 综合讨论
publishedAt: 2026-08-28
---

在 OpenClaw、Agent、MCP、插件和自动化实践里，Agent 最常见的翻车点不是“不会写命令”，而是“不知道这台机器上有什么、放在哪、用哪个版本”。于是它开始猜：`/usr/local/bin/python3` 不行就换 `C:\Python311`，`pnpm` 找不到就乱装，环境变量没加载就报认证失败。与其每次在对话里纠正，不如用一份 `tools.md` 把本地环境契约固定下来。

## 背景

Agent 调用本地工具时，依赖三件事：命令是否存在、路径怎么解析、运行环境是否就绪。不同机器差异很大，尤其是 macOS 用 Homebrew、Windows 用 Scoop/nvm-windows、Linux 用 apt/pyenv。把这类信息写死在 system prompt 或 README 里，很快会过期，而且容易把密钥、绝对路径一起带进版本库。

## 问题

常见的坏味道包括：

- 环境信息散落在 system prompt、README、脚本注释里，Agent 不知道该信哪个。
- 写了绝对路径，换机器就失效。
- 把 token、私钥直接写进工具说明。
- 文档越长越没人看，Agent 上下文窗口有限，超过两百行后基本被忽略。
- MCP server 的启动环境与 shell 工具混在一起，排障困难。

## 做法/步骤

### 1. 建立分层 tools.md

建议项目级和用户级分开：

- 项目级：`./tools.md`，描述当前项目需要的工具链。
- 用户级：`~/.config/openclaw/tools.md`，描述这台机器通用的环境约定。

Agent 启动时优先读项目级，再读用户级作为兜底。

### 2. 使用最小模板

每个工具只写五类信息：用途、查找方式、环境变量、验证命令、平台差异。例如：

```markdown
## node
- What: 运行 JS 脚本、启动 MCP server
- Where: 优先使用 fnm/nvm 管理的版本，不要使用系统自带 Node 10
- Check: command -v node && node -v
- Env: NODE_ENV=development
- Test: node -e "console.log(process.version)"
- Notes: 最低 Node 18；Windows 下用 nvm-windows
```

这里的关键是：`Check` 用 `command -v` 或 `which` 做运行时探测，而不是写死路径。Agent 可以先执行检测，再决定调用哪个可执行文件。

### 3. 只记录“已验证可执行”的命令

不要写“可能”“应该”“大概”这样的模糊描述。每一条都应该是你在当前机器上实际跑通过的。如果换成新机器没验证过，就标记为“未验证”。

### 4. 环境变量声明与值分离

`tools.md` 只声明“这个工具需要哪些变量”，不写实际值。实际值放在 `.env`、系统钥匙串或 CI 变量里。例如：

```markdown
## aws-cli
- Env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION
- Check: command -v aws && aws sts get-caller-identity
- Notes: 密钥不要写入本文件
```

配合 `direnv` 或 `.env` 加载机制，可以保证 Agent 每次执行前环境是干净的。

### 5. MCP server 单独一节

stdio 类型的 MCP server 对启动环境很敏感。可以这样写：

```markdown
## mcp-server-git
- Type: stdio
- Command: npx -y @some/mcp-server-git
- Env: GITHUB_TOKEN, GIT_AUTHOR_NAME
- Health: 启动后调用 tools/list，确认 git 相关工具可见
- Notes: 需要先确保 node 环境可用；Windows 下使用 cmd /c 包装
```

## 踩坑点

- **非交互 shell 不加载配置**：Agent 执行命令时经常拿不到 `.bashrc`、`.zshrc` 里的 PATH。解决办法是让 Agent 使用 login shell，或在 `tools.md` 里明确写“如找不到命令，先执行 `source ~/.zshrc` 或显式 export PATH”。更稳的是直接使用绝对路径，但只在用户级 `tools.md` 里维护，不要进项目仓库。
- **绝对路径陷阱**：项目级文档里写 `/home/alice/.nvm/...`，换到 Bob 的机器就失效。用 `command -v` 加 fallback 顺序，比写死路径可靠。
- **密钥泄漏**：任何文档、截图、日志里出现真实 token，都是事故。`tools.md` 里只写变量名，不写值，并加入 `.gitignore` 规则忽略 `.env.local`。
- **版本漂移**：同一工具不同版本参数不兼容。标明最低版本，并给出验证版本命令。
- **文档膨胀**：Agent 不是人类读者，它不会逐字读长篇大论。保持每个工具五到八行，超过部分拆到独立文档。

## 可复用建议

1. 把 `tools.md` 纳入版本控制，但忽略本地覆盖文件，例如 `tools.local.md` 可以用于个人临时调整。
2. 在 CI 或初始化脚本里跑一遍所有 `Check` 和 `Test`，保证文档与当前机器一致。
3. 给 Agent 的 system prompt 注入一句：“在执行任何本地命令前，先阅读并遵守 tools.md 的环境约束；若工具未在 tools.md 中列出，先运行检测命令确认可用性。”
4. 每次新增工具、切换版本或发现新的平台差异时，顺手更新对应条目。文档过期比没有文档更危险。

## 总结

`tools.md` 不是工具使用手册，也不是环境配置的百科全书。它的定位是一份“环境契约”：机器提供什么工具、Agent 如何找到它、怎么验证它能用。保持简短、可执行、可验证，比堆砌内容更有复现价值。把本地环境差异关进 `tools.md`，Agent 就不会每次都在错误路径上浪费 token 和时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/128a0623b159ec33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c79bdb37a1865a40.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/72f27fc60427c3ad.png)

