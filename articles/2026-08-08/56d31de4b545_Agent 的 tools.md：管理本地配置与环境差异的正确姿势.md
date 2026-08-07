---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 32049
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在 OpenClaw、MCP 插件和各类 Agent 自动化实践中，工具链的定义常被收纳进一份 `tools.md` 文件。它最初可能只是工具名称和命令的清单，但随着团队协作和环境复杂度上升，`tools.md` 很快演变成承载本地配置、环境变量映射和路径约定的关键节点。

问题在于，很多项目把 `tools.md` 当作静态文档来写，导致同一份文件在开发机、CI 容器和生产环境之间频繁出现“工具未启动”“API key 未设置”“找不到命令”等无头错误。根本原因不是工具本身坏了，而是**本地配置和环境差异没有被系统性地管理**。

## 问题拆解

一个典型的 MCP 工具定义可能长这样：

```yaml
- name: brave-search
  command: npx
  args:
    - "-y"
    - "@modelcontextprotocol/server-brave-search"
  env:
    BRAVE_API_KEY: "${BRAVE_API_KEY}"
```

看起来清晰，但实际使用时你会遇到三类坑：

1. **变量缺失或为空**：`tools.md` 被提交到 Git，里面的占位符被原样留给下一个检出者，工具静默失败，日志里可能只有一句`Error: API key is required`。
2. **路径与环境绑定**：直接写 `/Users/me/project/data` 或 `C:\tools\bin`，换一台机器或 CI 跑起来就崩。
3. **环境切换无约束**：开发、预发、生产各自需要不同的 key 或地址，但 `tools.md` 只有一个版本，只好手动改来改去，容易出错更易泄露生产密钥到本地仓库。

## 做法：让 tools.md 变得“可解析”

核心思路是把 `tools.md` 从一份硬编码清单升级为**带变量、可接入环境来源的声明式配置**。下面给出一个经过验证的步骤方案。

### 1. 建立环境变量契约

为每个需要外部注入的值定义一个清晰的变量名，并在 `tools.md` 中使用 `${VAR_NAME}` 引用，而不是写死值。同时提供 `tools.example.md` 或 `.env.example`，列出所有必填变量和示例：

```bash
# .env.example
BRAVE_API_KEY=your_api_key_here
DB_PATH=./data/local.db
TOOL_HOME=${HOME}/.local/bin
```

这样做既避免敏感信息泄漏，又让新加入的开发者一目了然。

### 2. 利用 profiles 切分环境

当工具在不同环境需要不同指令或不同参数时，可以在 `tools.md` 中引入简单的 profile 概念。例如用 YAML 的多文档结构或 `default/overrides` 字段：

```yaml
tools:
  - name: local-db
    command: sqlite3
    args: ["${DB_PATH}"]
profiles:
  production:
    tools:
      - name: local-db
        command: pgcli
        args: ["${DB_URL}"]
```

Agent 启动时根据环境变量 `ENV=production` 合并对应 profile，既保留了单文件管理的便利，又避免了分支化。

### 3. 统一加载入口

不要让每个开发者自己去想怎么加载 env。在项目根目录放置启动脚本 `run-agent.sh` 或 Makefile，强制按顺序执行：

```bash
#!/bin/bash
set -a
[ -f .env ] && source .env
[ -f .env.local ] && source .env.local  # 本地覆盖，不提交
# 然后启动 Agent / OpenClaw 进程，由它解析 tools.md
```

同时，`tools.md` 解析逻辑应能自动展开环境变量（例如使用 `envsubst` 或运行时替换），并检查必填变量是否缺失。缺失时直接报错停止，不要静默回退。

## 踩坑实录

- **占位符未替换直接被传给子进程**  
  某些 MCP 服务器启动时会校验环境变量格式，如果收到字面串 `${KEY}` 而不是实际值，可能不报错但功能异常。务必在交给子进程前由 Agent 完成全部变量替换，不要依赖 shell 的延迟展开。
- **路径分隔符在不同操作系统下的坑**  
  如果 `tools.md` 里使用了 Windows 风格路径 `C:\data`，在 Linux CI 中会因反斜杠被视为转义符而出错。统一使用正斜杠 `/`，或运行时用 `path.normalize()` 处理。
- **env 字段覆盖与系统环境变量冲突**  
  当 `tools.md` 中的 `env` 设计为“追加”而非“覆盖”时，宿主机已有的同名变量可能泄漏给工具，导致行为不一致。明确规则：`tools.md` 内的 `env` 应作为最低优先级的默认值，还是最高优先级？团队内部需要达成一致并写进文档。
- **.env.local 没被 .gitignore 拦下**  
  即使设计了覆盖文件，也容易出现将本地真实密钥误提交的意外。强制在 CI 中加入检查：`git ls-files --error-unmatch .env.local`，如果存在就阻断。

## 可复用建议

1. **提供校验工具**：编写 `validate-tools.sh`，先读取 `.env.example` 提取变量名，再确认当前环境全部存在，非空。可以在 `tools.md` 注释中标注 `# required`。
2. **模板化仓库**：维护一份 `tools.md.tmpl`，新项目通过 `sed` 或 `envsubst` 生成最终文件，避免手动复制。
3. **利用 direnv 自动加载**：让开发者在进入项目目录后自动注入环境变量，减轻记忆负担。
4. **敏感信息分级**：对于无法避免写进 `tools.md` 的路径（如证书文件位置），使用 `${HOME}/.ssh/cert` 这样的通用变量，而非绝对路径。
5. **GitHook 检查**：配置 pre-commit hook 扫描 `tools.md` 中是否残留 `${}` 以外的明文密码模式（如 `token=sk-`) ，防止意外提交。

## 总结

`tools.md` 不是一次性写完就束之高阁的静态文档，而是 Agent 本地配置体系中最容易工程化的一环。通过变量注入、profile 切换和统一的加载策略，可以把环境差异从“每次调试的荆棘”变为“可预期的参数输入”，让工具在多台机器、多个环境之间稳定运行。

真正面向工程的 `tools.md` 实践，不在于把配置写得多花哨，而在于任何人拿到仓库后，遵循一次 `cp .env.example .env` 和一次 `./run-agent.sh` 就能零报错地跑起来。

---

