---
title: Agent 的 tools.md：把本地环境差异变成可执行上下文
feedId: 34960
source: 综合讨论
publishedAt: 2026-08-28
---

# 背景：为什么 agent 总在本地环境上栽跟头

Agent 在执行本地任务时，真正难倒它的往往不是业务逻辑，而是环境假设。比如项目明明用 uv 管理依赖，agent 却反复尝试 pip install；虚拟环境在 `.venv`，它调用了系统 Python；README 写着开发命令，但 agent 不会稳定地从几千字中提取有效信息；有的机器用 `python`，有的用 `python3`，Windows 还要处理 PowerShell 激活脚本。这些问题散落在 README、Makefile、dotfiles、CI 配置和开发者脑子里，Agent 拿不到一个稳定的“可执行上下文”。

# tools.md 解决什么问题

tools.md 不是替代 README，而是给 Agent 和自动化任务提供一份靠近项目的本地环境索引。它回答几个问题：本机有哪些工具、版本约束是什么、如何激活环境、常用命令入口在哪、不同平台有什么差异、如何验证环境可用。把这部分信息从长文档中剥离出来，能显著减少 Agent 的试错次数，也方便插件和 MCP 调用前做前置校验。

# 做法：维护一份最小可用的 tools.md

建议放在仓库根目录或 `.agent/tools.md`，并在 Agent 的项目规则或系统提示中明确要求“执行本地命令前先读取 tools.md”。

内容分块保持固定：

1. 环境与包管理
2. 常用命令入口
3. 本地工具清单
4. 路径约定
5. 平台差异
6. 验证命令

示例片段：

```markdown
# tools.md

## 环境
- Python 3.11+，虚拟环境在 `.venv`
- 依赖管理：uv
- 激活：`source .venv/bin/activate`（Linux/macOS）
- 激活：`.venv\Scripts\Activate.ps1`（Windows）

## 常用命令
- 测试：`uv run pytest -q`
- 类型检查：`uv run mypy src`
- 启动开发服务：`uv run uvicorn app.main:app --reload`

## 本地工具
- `just`：统一任务入口，安装 `brew install just` / `scoop install just`
- `git-crypt`：如启用，由 pre-commit 自动处理

## 验证
- `uv run python -c "import sys; assert sys.version_info >= (3, 11)"`
- `just --version`
```

关键点是：tools.md 只描述实际存在的工具和入口，不写“推荐使用什么”或大段解释。它的读者首先是 Agent，其次是人。

# 踩坑点

- **信息过时**：tools.md 写一次就不再更新，很快会误导 Agent。建议把验证命令纳入 CI 或 `make verify`，当环境变化时能显式暴露出文档滞后。
- **与真实执行脱节**：文档写 pip，实际用 uv/poetry；文档写 `npm test`，实际需要 `npm run test:unit`。维护时以 Makefile、justfile、package.json、pyproject.toml 等可执行文件为准。
- **塞得太多**：把工具教程、架构说明、业务背景都放进 tools.md，会污染上下文，让 Agent 找不到重点。只保留差异和入口，通用知识不必重复。
- **泄露敏感信息**：绝对不要写 token、私钥、内部地址、用户名密码。非敏感的环境变量名可以写，值不要写。
- **平台差异凭印象写**：Windows 路径、激活脚本、PowerShell 语法要实际验证，否则会在异机执行时翻车。
- **忽略工具缺失场景**：不是每台机器都装了 `just` 或 `git-crypt`，应给出检测命令或安装提示，而不是假设已存在。

# 可复用建议

- 把 tools.md 拆成“稳定部分”和“自动生成部分”。工具版本、脚本入口可以由脚本从 lock 文件或 package.json 中提取生成，避免手写漂移。
- 给 tools.md 配一个验证命令：`make verify-tools` 或 `just verify` 逐个执行文中的验证命令，失败即提示更新。
- 固定命名，如 `tools.md` 或项目约定的 `AGENTS.md`，让 Agent、插件和人都能按路径找到。
- 工具链变化时，把 tools.md 纳入 PR 检查：改动 Makefile、package.json、poetry.lock、mise.toml 等，就要顺手检查文档是否需要同步。
- 如果项目同时提供 MCP server 或本地插件，tools.md 可以记录启动 MCP 所需的本地依赖和验证命令，但不要写入 MCP 的 schema、密钥或动态令牌。
- 与 Agent 逻辑解耦：tools.md 只写事实，不写“当运行测试时你应该先激活 venv 再执行 pytest”这类推理，推理交给 Agent 或规则层，避免绑定特定模型。

# 总结

tools.md 的本质是把“本地配置和环境差异”压缩为 Agent 可验证、可执行的最小上下文。它不追求面面俱到，而是追求每次执行前都能快速确认：环境怎么激活、工具在哪、命令怎么跑、如何验证。保持短小、可验证、与真实工具链一致，才能真正减少 Agent 的环境试错，让本地自动化任务更稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b7d3c1529b0e0240.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/572352ede4c35c3d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/def152b8988b430c.png)

