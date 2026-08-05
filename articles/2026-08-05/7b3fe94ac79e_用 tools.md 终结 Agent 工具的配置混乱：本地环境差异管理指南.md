---
title: 用 tools.md 终结 Agent 工具的配置混乱：本地环境差异管理指南
feedId: 31773
source: 综合讨论
publishedAt: 2026-08-05
---

# 用 tools.md 终结 Agent 工具的配置混乱：本地环境差异管理指南

## 背景：当 Agent 开始依赖本地环境

Agent 不再是纯对话模型。在 OpenClaw 这类智能体系统中，Agent 经常直接调用本地工具：读写文件、执行 Shell 命令、通过 MCP 桥接外部 API。这些工具依赖路径、凭据、网络代理等本地配置。一个典型的例子：开发者在 macOS 上写了一个“截图并发送到企业微信”的工具，它依赖 `/usr/sbin/screencapture` 和 `~/.wecom_token`。当这个工具被搬运到同事的 Linux 机器或 CI 的容器里，路径全错，Token 也找不到，Agent 直接报错退场。

环境的差异性是工程事实。问题不在于差异本身，而在于我们如何把这些差异表达出来，让 Agent 能可靠地适配，而不是把配置硬编码进 Python 脚本或 bash 函数里。

## 问题：散落的配置变成 Agent 的隐形债务

大量 Agent 工具的配置以几种不安全的形式存在：

- **硬编码路径**：`/home/alice/projects/data/`，换一台机器就失效。
- **环境变量依赖但未声明**：工具代码直接 `os.getenv("DB_URL")`，不提供默认值或文档。
- **敏感信息混入命令**：在 `shell_command` 里明文写 Token，既难切换环境，也极易泄露到日志。
- **平台绑定**：假定分隔符为 `/`，在 Windows 下调用相同的 Agent 工具会直接崩溃。

这些问题的根因是**工具配置没有被当作一等对象来管理**。tools.md 正是为了改变这一点——通过一份可读、可解析、可版本控制的结构化文档，把 Agent 工具的本地依赖集中收纳起来。

## 做法：用 tools.md 构建配置清单

tools.md 不是配置文件格式标准，而是一种约定：在 Agent 的 `tools/` 目录下放置一个 Markdown 文件，它同时承担文档和配置声明的角色。我们利用了 Markdown 的 YAML frontmatter 和代码块，让人类和机器都能读懂。

### 步骤 1：定义配置块

```markdown
---
# tools.md
profile: local
env:
  DATA_DIR: ${HOME}/.openclaw/data
  LOG_LEVEL: info
paths:
  screenshot_tool: /usr/bin/screencapture
  ffmpeg: ${FFMPEG_PATH:-/usr/local/bin/ffmpeg}
secrets:
  wecom_token: env:WECOM_TOKEN
---

# 截图工具

依赖系统命令 `screencapture`（macOS）或 `gnome-screenshot`（Linux）。
根据 profile 自动选择路径。
```

这里的关键设计：
- 占位符 `${HOME}` 由 Agent 启动时解析为当前用户的家目录。
- `${VAR:-default}` 提供带默认值的环境变量回退。
- `secrets` 映射不直接写值，而是引用环境变量，避免提交到 Git。

### 步骤 2：支持多环境 profiles

当需要区分开发机和 CI 容器时，可以在 tools.md 中使用条件块：

```markdown
## 环境配置

<!-- profile:local -->
DATA_DIR: ./workspace/data
CACHE_DIR: /tmp/cache

<!-- profile:ci -->
DATA_DIR: /runner/_work/data
CACHE_DIR: /tmp/cache
```

Agent 启动时传入 `--profile=ci`，配置加载器会只解析对应注释下的键值。这样一套工具可以在多个环境间无缝切换，不需要修改任何工具代码。

### 步骤 3：集成到 Agent 生命周期

在 OpenClaw 的 Agent 启动脚本（或工具调用入口）中，增加一个轻量解析器：

1. 读取 `tools.md`，抽取 frontmatter 或标记的配置。
2. 替换变量，注入当前进程的环境变量（仅当工具执行时）。
3. 将路径与密钥以只读方式暴露给工具函数，而不是依赖全局 `os.environ`。

这样做的好处是，工具函数可以直接 `from config import get_tool_config` 或通过 Agent 上下文获取配置，而不再自行探测环境，职责更清晰。

## 踩坑点与工程化避雷

在落地过程中，我们遇到了几个典型的坑，值得提前准备：

1. **路径分隔符地狱**  
   不要在 tools.md 里写死 `/`。使用 `pathlib` 或让解析器根据平台动态替换。可以采用 `${SEP}` 占位符，由加载器注入 `os.sep`。

2. **空值风险**  
   如果 `WECOM_TOKEN` 环境变量未设置，`env:WECOM_TOKEN` 会解析为空字符串。一定要提供清晰的校验报错。我们在解析阶段添加了 `required` 字段：  
   `wecom_token: env:WECOM_TOKEN required: true`  
   如果缺失则直接阻止 Agent 启动，而不是到工具运行时才报错。

3. **敏感信息日志泄露**  
   Agent 执行命令时常常会把全量环境变量序列化进日志。tools.md 的 secrets 字段应被标记为“非可打印”，加载器返回的配置对象在 `__repr__` 中隐去值。

4. **tools.md 与 tools.local.md 冲突**  
   团队协作时，本地覆盖可以放在 `tools.local.md` 并加入 `.gitignore`。解析器优先读取 local 文件，覆盖共享配置中的同名键。这样可以兼顾默认行为和个性化调试。

## 可复用建议

把这种模式固化到你的 Agent 工程里，可以遵循以下清单：

- **所有工具目录下都要求有 tools.md**：即使只有一个 shell 命令，也要写明依赖的二进制和环境变量。
- **使用 `.env` + `tools.md` 的组合**：`.env` 保存本地密钥（不提交），`tools.md` 声明哪些环境变量被用到，并提供开发环境的默认值。
- **提供配置校验脚本**：在 Agent 启动前运行 `config-check`，验证所有 `required: true` 的项都能解析成功。
- **CI 中注入 profiles**：通过环境变量 `PROFILE=ci` 让 Agent 自动选取正确的路径和设定，避免手动修改文件。
- **将 tools.md 作为工具文档的一部分**：当 Agent 出现“找不到命令”的错误时，首先查阅 tools.md，而不是直接去读代码。

## 总结

tools.md 并不是银弹，它只是一种减少配置债务的工程约定。在 Agent 工具增多的过程中，如果我们不主动管理环境差异，最终会收获一个在每台机器上行为都不一样的脆弱的智能体。用一份结构化文档把依赖、路径、凭据集中起来，再通过轻量解析器按需注入，这种做法能显著降低调试成本，也让团队间的工具共享真正可行。

下次当你准备在 Agent 工具里直接写死 `/opt/homebrew/bin/ffmpeg` 时，停下来想一想：把它写进 tools.md，给自己和未来的接手者一个交代。

---

