---
title: Agent 的 tools.md：让本地配置和环境差异有据可查
feedId: 33905
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 这类 Agent 项目里，工具调用早就不再是框架内部写死的几个函数。更多时候，Agent 依赖的是本机真实环境：Python/Node 解释器、浏览器路径、数据库客户端、MCP server、本地代理、CLI 凭证等。同一份 agent 配置，从笔记本换到开发机，或者从开发机部署到服务器，经常出现“逻辑没问题，但工具调不起来”。

这类问题通常不是代码 bug，而是环境差异。它们分散在 `.env`、shell profile、MCP 配置、README 甚至口头同步里，排查成本很高。`tools.md` 可以作为一份“环境契约”，把这些差异显式管理起来。

## 问题

常见的 tool 调用失败，往往来自这些差异：

- `python3` 和 `python` 指向不同版本，或者 `pipx`/`uv` 安装的 CLI 不在 PATH；
- MCP server 启动命令在另一台机器不存在；
- 本地代理端口不一致，导致 fetch 超时；
- 用户目录、数据目录、临时目录路径不同；
- 某些 CLI 需要预先登录，但凭据来源没有说明；
- shell 登录环境与非登录环境变量不一致，systemd 或 agent 进程读不到。

如果每次都靠报错再排查，等于把环境维护成本转嫁给了每一个协作者和每一次部署。

## 做法

建议把 `tools.md` 放在仓库根目录或 `.openclaw/` 下，作为工具依赖与环境差异的唯一说明文件。

一个可用的结构可以是：

```markdown
# tools.md

## 适用环境
- hostname / OS / shell / 是否容器

## 工具链版本与路径
| 工具 | 版本 | 安装方式 | 可执行路径 | 备注 |
|------|------|----------|------------|------|
| python3 | 3.12 | uv python install | `which python3` | 不用系统 Python |
| node | 20.x | nvm | `which node` | 部分 MCP 依赖 |
| chromium | 126 | playwright install | `which chromium` | 无头模式 |

## MCP/插件配置
- server 名称、启动命令、所需环境变量、监听端口

## 环境变量与非敏感配置
- 代理端口、数据目录、日志目录、非敏感开关

## 自检
运行 `scripts/check_tools.sh`，结果应与上表一致。
```

具体规则建议：

1. **只记录非敏感信息**。API key、token、密码一律使用占位符，指向 `.env` 或 secret manager。
2. **优先写命令名，不写绝对路径**。例如写 `uvx` 而不是 `/Users/xxx/.local/bin/uvx`，除非该路径本身就是关键差异。
3. **按环境分节**。例如 `### macos-local`、`### linux-dev`，不要把多台机器混在一起。
4. **变更环境先改文档**。配置文件调整后，`tools.md` 和实际配置要在同一个 PR 里出现。

再配合一个自检脚本，例如 `scripts/check_tools.sh`，自动检查命令是否存在、版本是否匹配、MCP server 能否启动、端口是否监听。脚本输出与 `tools.md` 对照，减少人工判断。

## 踩坑点

- **只记不更新**：文档很快失真，最后比没有文档还误导。
- **把 secret 写进 tools.md**：即使仓库私有，也不应该把真实 key 提交进去。
- **只写工具名，不写安装方式**：别人知道需要 `uvx`，但不知道是通过 `uv` 还是 `pipx` 安装。
- **忽略 shell 差异**：bashrc 和 zshrc 加载逻辑不同，交互式 shell 能跑，不代表 agent/systemd 进程能跑。
- **MCP 配置写死个人路径**：例如 `/Users/xxx/.local/bin/...`，换个人就失效。
- **以为 `.env` 能覆盖一切**：有时候系统级环境变量优先级更高，或者 agent 进程根本没加载 `.env`。

## 可复用建议

- 先维护一个模板，例如 `docs/tools.template.md`，新环境直接从模板生成。
- 写一个采集脚本，把 `uname -a`、`which python3`、`python3 --version` 等输出自动填入工具链表，减少手抄错误。
- 在自检结果里用状态标记：✅ 可用、⚠️ 版本偏差、❌ 缺失，不要手写状态，由脚本生成。
- 多机器场景可以使用目录维护：`docs/tools/macos-local.md`、`docs/tools/linux-dev.md`。
- 把 `tools.md` 纳入贡献前置检查，配置类 PR 必须同步更新文档。

## 总结

`tools.md` 不是 README 的装饰品。它更像 Agent 工具链的“环境契约”：把机器差异、版本差异、路径差异和启动依赖显式化。维护好它，不是为了文档好看，而是为了让“我这能跑”变成“按文档能复现”。对于 OpenClaw、MCP、插件和自动化实践来说，这比多写几个工具说明更有长期价值。

---

