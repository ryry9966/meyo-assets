---
title: Agent 的 tools.md：把本地环境差异变成可检查的配置
feedId: 35050
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 这类 Agent 实践中，本地工具调用经常出现：同一份 Agent 配置，在 A 机器能跑，在 B 机器因为路径不同、工具未安装、环境变量缺失而失败。tools.md 可以作为 Agent 读取的“本地能力清单”，把命令、参数、环境要求和平台差异显式化，减少隐式假设。

## 问题

常见问题不是“不会写配置”，而是把 tools.md 写成了“一次性脚本备忘录”：

- 硬编码绝对路径，换机器就崩。
- 把 token、私钥直接写在 tools.md 里。
- 假设某个 CLI 一定在 PATH 中。
- 只覆盖 macOS，忽略 Windows 和 Linux 的差异。
- tools.md 和实际代码中的工具定义不同步。
- 环境变量只在当前 shell 存在，Agent 启动后读不到。

这些问题会让 Agent 的“本地执行”变成黑盒，排障时只能靠猜。

## 做法/步骤

### 1. tools.md 只写契约，不写秘密

每条工具描述应包含：

- `name`：工具名
- `description`：给 Agent 看的用途说明
- `command`：命令模板，使用占位符
- `args`：参数列表或模板
- `env`：需要读取的环境变量名，不写值
- `platforms`：支持的平台，如 `["darwin", "linux", "windows"]`
- `requires`：依赖的本地工具或文件
- `fallback`：失败时的降级策略

示例片段：

```markdown
## git_status
- command: git
- args: ["status", "--short", "--branch"]
- requires: git >= 2.30
- platforms: [darwin, linux, windows]
- env: []
```

密钥类信息通过环境变量引用，例如 `env: ["GITHUB_TOKEN"]`，值由启动脚本或外部注入，绝不落盘到 tools.md。

### 2. 分层配置，避免全局污染

建议拆成三层：

- 全局 `tools.md`：通用工具，如 `git`、`rg`、`python`。
- 项目级 `tools.md`：仅当前仓库需要的 CLI、脚本。
- 用户级覆盖 `tools.local.md`：个人机器上的路径修正、额外工具，不入库。

加载顺序为：全局 → 项目级 → 用户级覆盖。后者可以覆盖前者的字段，但禁止覆盖 `command` 的安全边界。

### 3. 处理环境差异

不要写死路径。用以下方式解耦：

- 命令使用 `PATH` 查找，不写 `/usr/local/bin/xxx`。
- 对 Windows 单独列出 `command` 或 `shell` 字段，例如 `powershell -Command ...`。
- 在 tools.md 中提供 `setup` 或 `bootstrap` 字段，指向安装脚本，例如 `scripts/setup_linux.sh`。
- 使用 `preflight` 检查：启动前运行 `tools.md check`，逐条验证工具是否存在、版本是否满足、环境变量是否已设置。

### 4. 保持同步

tools.md 不应该手写后束之高阁。建议在代码中维护工具定义，生成 tools.md；或者至少让 CI 跑一个校验，检查 tools.md 中声明的 `command` 是否能在干净容器里通过 preflight。

## 踩坑点

- **环境变量空值**：Agent 启动时通过 systemd / launchd / Docker 等环境不同，可能读不到交互 shell 的 `export`。解决：在启动脚本里显式加载 `.env`，并打印遮蔽后的值确认。
- **相对路径基准**：很多本地工具依赖当前工作目录。Agent 执行时 cwd 可能是项目根目录，也可能是临时目录。tools.md 里应明确 `cwd` 策略，或在命令中显式传入路径。
- **平台分支过细导致难维护**：不要为每个平台写完全独立的命令。优先用跨平台工具（如 Python 脚本、Node CLI）替代 shell 差异。
- **权限问题**：本地工具可能需要读写某个目录。tools.md 应记录需要的权限或提供 `--dry-run` 模式。
- **把 tools.md 当文档而非配置**：如果 Agent 不读取它，它就没有意义。确保加载逻辑简单可靠，最好在启动日志里打印实际生效的工具数量。

## 可复用建议

1. 模板化：提供一个最小 tools.md 模板，包含 `preflight` 和常用工具条目。
2. 隐藏敏感值：所有密钥通过环境变量注入，启动时输出脱敏信息。
3. 版本固定：在 `requires` 中写最低版本，避免“能用但行为不同”。
4. 用同一份 tools.md 做 CI 检查：在 CI 中用干净镜像运行 `preflight`，确保新增工具已被声明。
5. 记录平台差异在单独文件：例如 `tools.windows.md`，通过条件加载，不让主文件膨胀。

## 总结

tools.md 的价值不是“把命令记下来”，而是把本地环境差异从 Agent 的隐式假设变成可检查、可降级、可复现的配置。先解决密钥不落盘和平台解耦，再逐步引入分层加载和 preflight，能显著减少“我这能跑，你那不行”的问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2e92298d10916cd0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/5c65bf3b4a234af8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f8826e12bf7f8ddb.png)

