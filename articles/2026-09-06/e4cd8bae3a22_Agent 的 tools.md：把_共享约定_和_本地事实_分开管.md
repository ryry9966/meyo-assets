---
title: Agent 的 tools.md：把"共享约定"和"本地事实"分开管
feedId: 36328
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

跑同一套 OpenClaw 配置的人通常不止一台机器：日常笔记本、家里的服务器、一台 VPS。workspace 里的 AGENTS.md、skills、插件配置都进 git 同步，但 agent 真正干活时依赖的"本地事实"——shell 是什么、python 装在哪、有没有 ffmpeg、出网要不要走代理——每台机器都不一样。工具层（MCP server、CLI 插件）是统一的，环境不是。

## 问题

环境差异导致的失败很典型：agent 信心满满地执行 `pip install`，结果机器上是 conda；去找 `/opt/xxx/venv`，路径根本不存在；在 macOS 上跑 `sed -i` 少个参数。这些失败有共同点：不是模型能力问题，而是它对这台机器的认知全靠猜。

把机器事实写进 AGENTS.md 能缓解，但 AGENTS.md 要跨机同步，写死了 A 机器的路径会污染 B 机器；把凭证、本机路径这类信息提交进仓库更是直接事故。不写呢，agent 每次都要试错，token 和时间双输。

## 做法

思路是分层：**共享层管"怎么做事"，本地层管"这台机器有什么"**。tools.md 就是本地层。

1. **盘点**：写个一次性脚本 dump 环境事实——OS/shell、关键 CLI 的 `which` 结果和版本、常用路径、代理设置、GPU。
2. **落盘**：整理进 workspace 的 tools.md，固定分节：Runtime / 可用 CLI / 路径约定 / 网络与代理 / 已知限制。
3. **挂载**：在 AGENTS.md 里加一条指令，session 开始先读 tools.md，而不是把这些事实硬编码进 system prompt。
4. **隔离**：tools.md 进 `.gitignore`，仓库里只提交 `tools.example.md` 模板，各机器各自维护。
5. **维护**：agent 发现工具缺失或路径不符时，让它顺手更新 tools.md，把文档变成自维护的清单。

一个实际的模板长这样：

```markdown
# tools.md — 本机环境事实
> 更新：2025-06-12（盘点脚本生成）

## Runtime
- Debian 12 / zsh
- node 22.x (nvm)，python 3.12 系统级，无 conda

## 可用 CLI
- ffmpeg / rg / jq / sqlite3 ✓
- 无 docker，勿尝试

## 网络与代理
- 出外网走 http://127.0.0.1:7890

## 已知限制
- 磁盘余 8G，别下载大文件
```

## 踩坑点

- **别写密钥本体。** tools.md 虽然不进 git，但内容会被读进模型上下文，可能随日志外泄。写指针："凭证在 `~/.xxx`，通过 `ENV_NAME` 引用"，不写值。
- **过期信息比没有更糟。** agent 会信任 tools.md，里面写着的工具实际没了，排查方向直接被带偏。文件头加更新日期，并要求 agent 使用前抽查关键项。
- **别什么都往里塞。** `which` 十毫秒能查到的事不值得预写。tools.md 留给"发现成本高"的事实，比如这台机器 443 出不去、GPU 只有一张。
- **控制篇幅。** 它每个 session 都进上下文，几百行就是持续的 token 税，一屏以内最好。
- **分清层级。** 全局一份管机器事实，项目内一份管项目专属命令，别混在一个文件里。

## 可复用建议

- 一句口诀：共享层写约定，本地层写事实。
- 排查"在我机器上是好的"这类问题时，先 diff 两台机器的 tools.md，往往比翻日志快。
- 换新机器时，跑一遍盘点脚本 + 拷贝模板，十分钟就能让 agent 进入可用状态，比手工调教快得多。

## 总结

tools.md 的价值不在文件本身，而在把"环境认知"从 prompt 里的硬编码，变成一层可维护、可隔离、可自查的配置。agent 跨机器跑得稳不稳，往往就取决于这类不起眼的本地事实有没有被认真管理。先分层，再盘点，最后让 agent 自己维护它——这套做法成本很低，但回报是实打实的失败率下降。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/5ffd29401de5e1e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1946b4edd151cc02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/d8b224f395f3103f.png)

