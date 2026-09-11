---
title: 别把环境写进提示词：用 tools.md 管理本地配置与环境差异
feedId: 37055
source: 综合讨论
publishedAt: 2026-09-11
---

# 别把环境写进提示词：用 tools.md 管理本地配置与环境差异

## 背景

用 OpenClaw、MCP 工具链或各类本地 Agent 做自动化，时间长了会发现一个规律：任务逻辑大同小异，真正折腾人的是环境。Mac 上是 BSD 版 sed，Linux 上是 GNU 版；这台机器用 uv，那台还在 pip；公司内网走镜像源，家里要挂代理。Agent 并不天生知道这些。

## 问题：三种常见的错误姿势

1. **写死在系统提示词里。** 换台机器、升个版本，提示词就变成错误事实，Agent 照着执行只会翻车。
2. **让 Agent 每次自己探测。** 看似聪明，实则每次任务都要烧掉若干轮 `which`、`python --version`，试探过程中还可能误装包、误改文件。
3. **和环境说明混在项目文档里。** AGENTS.md 既写代码规范，又写“这台机器 Python 在哪”，职责糊了，谁也不敢删。

核心矛盾：环境知识是易变的、机器相关的，但 Agent 需要它稳定、可复现。

## 做法：一份分层的 tools.md

约定一个独立文件 `tools.md`，只写一件事：**这台机器上，Agent 干活要用到的环境事实**。

建议结构（可直接当模板）：

```markdown
# 运行时
- Python 3.12: /usr/bin/python3，包管理一律用 uv，不要 pip install
- Node 20（经 nvm）：先 source nvm 再跑 node

# 路径约定
- 缓存目录 ~/.cache/openclaw/，可随时清空
- 日志只写 ./logs/，不要碰 /var/log

# 网络环境
- 内网时使用镜像源 https://mirrors.example/pypi
- 代理仅在 https_proxy 已设置时可用

# 平台差异
- macOS：sed/grep 用 ggrep/gsed 替代
- Windows：一律走 WSL，不调 PowerShell 原生命令

# 禁区
- 不全局装包；不改 /etc/hosts；不动 ~/.ssh
```

落地四步：

1. **盘点**：跑一轮探测命令（`which -a`、版本号、`env`），把真实结果抄下来，不要凭记忆写。
2. **分层放置**：用户级放 `~/.openclaw/tools.md`（个人全局环境），项目级放仓库根目录（项目特有约定），项目级优先。
3. **挂载读取**：在 Agent 启动流程里显式加载该文件，而不是指望它跨会话“记得”。
4. **配套校验**：写个三十行的脚本，逐条 assert tools.md 里的声明（版本、路径、命令存在性），挂到 pre-commit 或 CI。事实一过期就报警。

## 踩坑点

- **写成愿望而不是事实。** “用 uv”写得很痛快，机器上根本没装 uv，Agent 照做直接报错。tools.md 的每一条都应该有校验背书。
- **贪多。** 这个文件会进上下文，写成长篇散文纯粹烧 token。控制在几十行，事实优先，解释性文字越少越好。
- **写了敏感信息。** 不要放 key、密码、内网拓扑明细。写“凭据从环境变量 XXX 读取”即可——文件本身是要进 git 的。
- **过期不更新。** 换机器、升依赖后不做校验，Agent 基于错误事实行动，比没有这个文件更危险——它会非常自信地做错事。

## 可复用建议

- 职责分清：**AGENTS.md 管“怎么干活”，tools.md 管“在哪干活、用什么干”**，互相不要渗透。
- 像 review 代码一样 review 环境变更：tools.md 进仓库，改动走 PR，谁动了环境谁更新文件。
- 多平台环境按 macOS / Linux / WSL 分节写，Agent 按当前平台取对应小节。
- 团队共性放项目级，个人偏好放用户级，避免把自己机器的怪癖强加给同事。

## 总结

tools.md 的本质，是把环境知识从“对话里的记忆”迁移成“声明式文件”：Agent 读到的永远是经过校验的当前事实，而不是上一次会话的残影。配置分层、事实可校验、变更可 review——做到这三点，同一个 Agent 在任何一台机器上的行为都是可预测的。文件不大，但它决定了 Agent 是在“猜环境”，还是“知道环境”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/99a5d634e6f1f119.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/8444a3c259dec9fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/847dd99c6c0ec1fd.png)

