---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 34680
source: 综合讨论
publishedAt: 2026-08-25
---

在 OpenClaw、Agent、MCP 和插件自动化项目里，`tools.md` 经常被当成“工具注册表”：告诉 Agent 本地有哪些命令、脚本、MCP 服务可以调用，参数怎么传，失败怎么处理。但实践中它很容易演变成一份本地环境快照，把 `/Users/xxx/project`、`python3.11`、某个盘符甚至密钥都写进去。结果同一份配置换到另一台机器、容器或 CI 环境后，Agent 不会先报告环境不匹配，而是拿着错误路径反复尝试，甚至自己拼一个“看起来差不多”的命令继续执行。

## 背景

`tools.md` 的初衷是降低 Agent 与本地工具之间的沟通成本。它应当描述“能力”，而不是记录“这台机器长什么样”。但很多项目在快速跑通后，把本地调试时的具体命令直接固化进文档。比如：

```markdown
## tool: compress_image
命令: /Users/me/bin/img_compress --quality 80
```

这种写法在作者机器上没问题，换到同事机器、容器、Windows 或 CI 环境后就会失效。Agent 读到的是一个不存在的路径，却不会自动理解“这是环境差异”，反而可能继续猜测 `/usr/local/bin/img_compress` 或 `img_compress.exe`，造成更隐蔽的错误。

## 问题

`tools.md` 与环境强耦合时，常见症状包括：

- Agent 执行失败后不断重试，浪费 token 和时间；
- 因为 `command not found` 触发幻觉，自己编造参数；
- 版本差异导致同一个工具在不同机器行为不一致；
- 密钥、token 被写进文档并提交到版本库；
- 有副作用的命令没有权限边界，Agent 可能在错误目录执行；
- MCP 服务启动命令写死 `npx`、`uvx` 或绝对路径，换环境后连接失败。

这些问题的本质是：`tools.md` 承担了它不该承担的环境职责。

## 做法

### 1. 拆成契约层和环境层

`tools.md` 只写工具契约：用途、参数、返回格式、副作用级别、预检命令。真实的可执行文件路径、Python 解释器、数据目录全部放到环境变量或 wrapper 脚本中。

例如：

```markdown
## tool: compress_image
- 用途: 压缩图片，支持 jpg/png
- 参数: input, quality
- 环境变量: IMG_COMPRESS_BIN, TMP_DIR
- 预检: bin/check-image-tool.sh
- 副作用: 只写输出文件，不修改原图
```

`IMG_COMPRESS_BIN` 的值在不同机器上可以不同，但契约不变。

### 2. 工具入口用 wrapper 脚本

不要直接让 Agent 调用系统命令。把差异封装到 `bin/tool-name` 脚本里，脚本内部根据 OS、架构或环境变量选择真实路径：

```bash
#!/usr/bin/env bash
if command -v cwebp >/dev/null 2>&1; then
  exec cwebp "$@"
elif [ -n "$IMG_COMPRESS_BIN" ]; then
  exec "$IMG_COMPRESS_BIN" "$@"
else
  echo "image tool not found; run bin/check-image-tool.sh" >&2
  exit 127
fi
```

`tools.md` 只引用 wrapper，不关心底层实现。

### 3. 所有可变值走环境变量

路径、端口、认证信息、临时目录、模型名等，都通过 `.env` 或配置管理。`tools.md` 中只写变量名，例如 `${DATA_DIR}`、`${FFMPEG_BIN}`。提交时提供 `.env.example`，真实 `.env` 不入库。

### 4. 为每个工具定义预检命令

给每个工具配一个 `healthcheck` 或 `preflight` 命令，比如 `bin/check-ffmpeg.sh`。预检只做只读检查：命令是否存在、版本是否满足、依赖目录是否可写。Agent 在调用工具前先跑预检，失败时直接返回结构化错误，而不是继续执行主命令。

### 5. MCP 服务也按工具管理

MCP 服务启动命令不要写死在 `tools.md` 或 Agent 配置里。用 wrapper 脚本启动，例如：

```bash
# bin/mcp-filesystem
if [ -n "$FILESYSTEM_MCP_BIN" ]; then
  exec "$FILESYSTEM_MCP_BIN"
elif command -v npx >/dev/null 2>&1; then
  exec npx -y @modelcontextprotocol/server-filesystem "$@"
else
  echo "MCP filesystem server not available"
  exit 127
fi
```

这样本地开发、容器部署、CI 环境可以各自提供不同的启动方式，而不需要改 `tools.md`。

## 踩坑点

1. **绝对路径写进 tools.md**：`/home/me/bin/...`、`C:\tools\...` 一换环境就坏。哪怕只是文档里的示例，也会误导 Agent。
2. **只写参数，不写前置条件**：Agent 不知道需要 `ffmpeg >= 6.0`，在旧版本上执行可能因参数不兼容而失败，然后继续尝试其他参数组合。
3. **密钥混入文档**：`API_KEY=xxx` 一旦提交，后续很难彻底清除。环境变量名可以出现在 `tools.md`，真实值永远不能。
4. **忽略 Windows 差异**：路径分隔符、换行符、盘符、`where` 与 `which` 的差异，都可能让 wrapper 在 Windows 上失效。尽量用跨平台脚本或明确标注“不支持 Windows”。
5. **没有副作用等级**：某些工具会删除文件、重启服务或修改系统配置。如果不在 `tools.md` 中明确副作用级别，Agent 可能在错误场景下调用，造成不可逆操作。

## 可复用建议

- **模板化**：为每个工具固定写六项：用途、参数、环境变量、预检命令、副作用级别、失败提示。
- **动态生成 tools.md**：写一个脚本探测本地环境，根据实际可用命令和版本生成 `tools.md`，减少手改成本。
- **用容器或版本管理锁定工具**：例如在 CI 中使用 `mise exec --`、`nix shell` 或 `docker run` 来提供稳定的工具版本。`tools.md` 只描述能力，版本锁定交给外部环境。
- **预检结果缓存**：预检可以输出 JSON，Agent 启动时只读结果，避免每次调用都重复检查。
- **提交 `.env.example`，不提交 `.env`**：真实环境差异只存在于本地，版本库里始终是可移植的契约。

## 总结

`tools.md` 的正确定位是“工具契约”，而不是“环境说明书”。把本地路径、版本、密钥和平台差异从 `tools.md` 中剥离出去，交给 wrapper 脚本、环境变量和预检命令处理，能显著减少 Agent 在跨环境下的错误重试和幻觉操作。工程上值得坚持一条原则：**凡是换一台机器就会变的东西，都不应该出现在 `tools.md` 里。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2673566f9c184b73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/7623094ef7a0f0fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8337b3767ab6eab3.png)

