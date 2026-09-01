---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35762
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

Agent 在本地执行工具调用时，对"这台机器"的了解几乎为零。它知道 MCP 怎么调，却不知道你的 node 是 fnm 还是 nvm 装的、Python 走 pyenv 还是系统自带、Docker 是 Desktop 还是 colima、哪些 MCP server 在这台机器上真的可用。这些事实散落在 `.zshrc`、项目 README 和你的脑子里。换机器、换人、换 Agent，它的第一反应是猜——然后猜错。

## 问题

实际项目里反复出现三类事故：

1. **路径硬编码**：配置写死 `/Users/xxx/...`，同步到别的机器直接失效；
2. **命令漂移**：Agent 用了旧命令，或者顺手 `npm i -g` 全局装包，污染环境；
3. **秘密混入**：为了让 Agent"能跑通"，把 token 写进配置，随仓库同步扩散。

根因不是 Agent 笨，而是环境知识没有被显性化、契约化。

## 做法

建立两层 tools.md：

- **全局层（机器级事实）**：OS、包管理器、语言版本管理方式、容器运行时、本机可用的 MCP server 列表；
- **项目层（repo 级约定）**：构建/测试命令、目录约定、禁止操作。项目层覆盖全局层。

骨架示例：

```markdown
## 环境
- macOS 15；node 由 fnm 管理，禁止 npm i -g
## 路径与数据
- 数据只写 ./data，禁用 /tmp
## 约定
- 测试：pnpm test
- 禁止：直接改 .env；未经确认执行 sudo
```

Agent 的入口配置只需一句："执行任何工具调用前，先读 tools.md。"

进阶一步：**事实部分不要手写**。写个 20 行的 bootstrap 脚本，输出 `which node`、`python3 -V`、`docker context ls` 的结果，把"机器快照"和"人写的约定"分开——快照过期可以随时再生。

## 踩坑点

1. **写成 wiki**。三千字的 tools.md 每次被整读，context 烧掉了，重点还被淹没。上限控制在 60 行左右。
2. **秘密进文件**。只写"secrets 在哪、怎么注入"，永远不写值本身。
3. **只写不复检**。环境升级后文件过期，Agent 照着错误信息执行，比不给信息更糟。把"核对 tools.md"加进升级 checklist。
4. **平台差异混写**。mac 与 Linux 的差异应分小节隔离，不要散落在同一节里让 Agent 自行分辨。

## 可复用建议

- 把 tools.md 定位为**机器快照 + 行为契约**，不是文档；
- 事实用脚本生成，人只维护约定；
- 约定部分进 git，机器相关的值用 `${HOME}` 之类占位符；
- 排障第一步永远是核对 tools.md 是否过期——它应该排在改 prompt 之前。

## 总结

tools.md 的价值不在文件本身，而在于把 Agent 对环境的隐性假设变成显性、可版本化、可复核的契约。写它只要半小时，省下的是每次换机器、每个新环境、每只新 Agent 的重复踩坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/bb132791bbdd6eb1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f30acaf1d0264196.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/173036b2ef1d3232.png)

