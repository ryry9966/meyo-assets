---
title: Agent 的 tools.md：把本地环境差异关进配置里
feedId: 35155
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：tools.md 是 Agent 的工具契约

在 OpenClaw/Agent 场景里，tools.md 不是把命令塞给模型背下来，而是给工具调用提供一份可审计的运行契约。工具可能是本地脚本、MCP server、插件 CLI，甚至是组合命令。问题在于，同一个项目在不同机器上，路径、运行时、密钥、端口、操作系统都可能不同。把这些差异散落在 prompt、脚本或开发者脑子里，Agent 就会时好时坏，调试成本很高。

## 问题：为什么工具“在你机器上能跑”

典型表现是：同事或 CI 机器上 Agent 调用 `python scripts/run.py` 失败，因为那台机器只有 `python3`；Windows 上路径写成 `/Users/xxx`；本地密钥直接写进 tools.md 被提交到仓库；端口 8123 被占用；或者 GUI 启动的 Agent 拿不到 shell 里的环境变量。根因不是 Agent 能力问题，而是工具配置没有区分“接口”和“本地实现”。

## 做法：把差异收敛到 tools.md

### 1. 盘点工具依赖
先列出每个工具需要的运行时、路径、密钥、网络、端口和操作系统差异。tools.md 顶部写清 assumptions，例如：

```yaml
assumptions:
  shell: bash
  os: [linux, darwin, windows]
  required: [python3, git, docker]
```

### 2. 用变量代替绝对路径
工具命令里不要出现 `/Users/tom/...` 或 `C:\Users\...`。统一用环境变量或项目变量：

```yaml
tools:
  - name: run_local_script
    command: ${PYTHON_BIN} ${WORKSPACE_DIR}/scripts/run.py
    env:
      PYTHON_BIN: ${PYTHON_BIN}
      LOCAL_API_KEY: ${LOCAL_API_KEY}
    timeout: 120s
```

真实值放在 `.env`、系统 keychain 或 CI secrets 中，tools.md 只声明变量名。

### 3. 区分平台变体
不要在一个 command 里硬塞 `if else`。给工具写 variants：

```yaml
  - name: open_browser
    platforms:
      darwin: open ${URL}
      linux: xdg-open ${URL}
      windows: start ${URL}
```

这比在 prompt 里教 Agent 判断系统更稳定。

### 4. 写 preflight 校验
每个工具最好带 `preflight` 或 `check` 命令，例如 `${PYTHON_BIN} --version`。Agent 在执行前可以运行校验，失败时直接收集环境信息，而不是盲目重试。

### 5. 与 MCP/插件配置打通
MCP server 的启动命令和环境变量也适合放进 tools.md 或统一的外部配置文件。避免同一份密钥在多处重复，导致改了一个地方另一个地方没改。

## 踩坑点

- **Windows 路径和空格**：`C:\Program Files\...` 不加引号会断裂，反斜杠在 YAML/JSON 里需要转义。建议用正斜杠或环境变量如 `%USERPROFILE%`。
- **GUI 启动拿不到环境变量**：macOS/Linux 桌面应用启动的 Agent 常缺少 shell profile 里的变量。可以在 tools.md 中显式加载 profile 或使用虚拟环境的绝对路径。
- **conda/venv 未激活**：不要依赖裸 `python`，写成 `${VENV_PATH}/bin/python` 或提供激活 wrapper。
- **密钥误提交**：tools.md 中出现真实密钥是高频事故。规则是 tools.md 只有变量名，敏感值放 `.env` 且加入 `.gitignore`。
- **固定端口冲突**：如果工具需要本地端口，用 `${MCP_PORT}` 或动态分配，避免 8123、3000 这类常用端口被占。
- **忽略超时**：拉 Docker、跑大脚本可能需要分钟级时间。给每个工具设置合理 timeout，并让 Agent 知道失败可能来自超时而非逻辑错误。

## 可复用建议

1. **把 tools.md 当接口文档**：只声明需要什么、输入输出、失败条件，不写具体实现。
2. **配套 `.env.example`**：tools.md 引用的所有变量都应在模板中列出。
3. **提供 `agent doctor` 工具**：让 Agent 自动运行 preflight，输出环境差异和修复建议，减少人工排查。
4. **版本控制策略**：提交 tools.md，但不提交 `.env`、本地绝对路径和生成的临时文件。
5. **收敛到 wrapper**：如果本地脚本很多，写一个统一入口脚本，tools.md 只调用 wrapper，减少配置膨胀。

## 总结

tools.md 的目标不是消灭环境差异，而是把差异显式化、可校验、可回滚。好的 tools.md 让 Agent 在陌生机器上先读契约再动手，而不是带着一把写死路径的钥匙到处试锁。把本地配置关进配置里，工具才会真的可复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/eb734e360cd909ca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a5c4fac176e41a3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a53dbe93c0e6f722.png)

