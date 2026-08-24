---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 34511
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景：Agent 为什么需要一份本地事实文件

很多 Agent 任务失败，不是模型不会做，而是执行环境判断错误。同一套提示在别人机器能跑通，换到你机器上就出现 `python: command not found`、`pnpm` 不存在、`ffmpeg` 路径不对，或者请求被本地代理拦下。Agent 常用的探测方式——`which`、`python --version`、`pip list`——既不可靠又有开销，有时还会产生副作用。与其让它反复试探，不如把本地环境写进 `tools.md`，作为动手前的“事实来源”。

## 问题：环境差异到底差在哪

典型差异包括：macOS 与 Linux 命令参数不同；Homebrew 在 Apple Silicon 和 Intel 机器上的路径不一样；Node/Python 由 nvm、conda、uv、pyenv 管理，但只有交互 shell 才加载；Windows 与 WSL 路径分隔符混用；企业代理、私有镜像、内部 registry；本地 wrapper 脚本替换了系统命令。这些信息如果不显式给出，Agent 很容易选错工具链，或者失败后盲目重试。

## 做法：把 tools.md 写成可执行的约束

### 1. 固定位置和读取约定

将 `tools.md` 放在 Agent 工作区根目录，并在项目说明或全局指令里明确：执行命令前先读 `tools.md`，按其中的路径和约束执行。不要只把它当成文档，要让它进入 Agent 每轮任务的前置上下文。

### 2. 先写机器身份

开头给出 OS、架构、shell、工作区路径，减少推断：

```text
host: macbook-pro-m2
os: macOS 14.5 (arm64)
shell: zsh
workspace: /Users/name/work/project
```

### 3. 列出关键工具和绝对路径

不要只写命令名，要用绝对路径消除歧义。示例：

| tool | path | notes |
| --- | --- | --- |
| python | /opt/homebrew/bin/python3.12 | 不使用系统 python |
| node | ~/.nvm/versions/node/v20.11.0/bin/node | 需先加载 nvm |
| ffmpeg | /opt/homebrew/bin/ffmpeg | 已安装 |
| uv | ~/.local/bin/uv | 已加入 PATH |
| docker | /usr/local/bin/docker | 需要先 `colima start` |

### 4. 记录环境变量与初始化命令

很多工具链依赖非交互 shell 不会自动加载的初始化脚本。明确给出执行前必须运行的命令，例如：

```bash
source .envrc
source ~/.nvm/nvm.sh
export PATH="/opt/homebrew/bin:$PATH"
```

不要假设 `.bashrc` 或 `.zshrc` 一定被加载。

### 5. 写清工具链偏好和禁用项

例如：Python 项目统一用 `uv run`，不要直接 `pip install` 到系统；Node 用 `pnpm`，不要全局装包；不要把 token、私钥路径、密码写进 `tools.md`，敏感信息只引用环境变量名。

### 6. 提供自检命令

给一组快速验证命令，让 Agent 在任务前先跑并比对版本：

```bash
command -v python3 && python3 --version
node --version
ffmpeg -version | head -1
```

## 踩坑点

- 路径包含空格却没有引号，导致命令解析失败。
- 只写 `python` 不写绝对路径，Agent 仍然可能调用系统版本。
- 非交互 shell 不加载 `.bashrc`/`.zshrc`，导致 nvm、conda、uv 找不到。
- Windows/WSL 混用，路径分隔符和换行符不一致。
- 把密钥、token 或内部地址写进 `tools.md` 后误提交。
- 文件太长，Agent 抓不住重点；建议控制在 80 行内。
- 绝对路径写死后换机器没更新，反而误导。

## 可复用建议

- 建立模板：host、os、shell、tools、env init、preferences、verify、notes。
- 要求 Agent 在开始前先复述环境和限制，再执行任务。
- 每次升级工具、切换 Python 版本、新增 MCP 工具后同步更新。
- 与 MCP 配置分离：`tools.md` 描述“本机怎么用”，MCP 配置描述“能调哪些外部能力”。
- 敏感信息一律放 secret manager 或环境变量。

## 总结

`tools.md` 是本地环境的适配层，不是装饰文档。它把 Agent 从“猜环境”切换到“读事实”，减少低级失误。保持简洁、结构化、可验证，是成本很低但收益稳定的工程习惯。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/9ca5f84200095243.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/d69e20ade4b13e22.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/954c125d64312e4b.png)

