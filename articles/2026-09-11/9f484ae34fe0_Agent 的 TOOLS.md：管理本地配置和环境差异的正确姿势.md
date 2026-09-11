---
title: Agent 的 TOOLS.md：管理本地配置和环境差异的正确姿势
feedId: 37019
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 的 agent 不是跑在真空里的：它要在本机执行 shell 命令、调用插件和 MCP 工具、操作浏览器与文件系统。模型自带通用知识，但你的机器是具体的——Homebrew 装在哪个前缀、Python 用哪个 venv、代理走哪个端口，这些模型一概不知。

workspace 里的 TOOLS.md 就是补这块的：一份写给 agent 看的"本机说明书"，随上下文加载，相当于把环境事实固化成 agent 可直接引用的契约。

## 问题

不写，agent 只能猜：猜路径、猜包管理器、猜版本，猜错就重试，费 token 也费时间。乱写更糟，常见三类问题：

1. **写成教程**：把通用 how-to 塞进去，与 skill 文档重复，上下文膨胀且很快过时。
2. **写完就忘**：系统升级、换机、改工具链后文件还是旧的，agent 按过期信息操作，比不写还危险。
3. **写进密钥**：token 直接落在文件里，随上下文进入模型，也可能随 workspace 同步外泄。

## 做法

原则一句话：只写"本机事实 + 验证过的命令"，一机一份，控制在百行级。骨架大致：

```markdown
# TOOLS.md — Mac Studio (macOS 15, arm64)
## Shell & 包管理
- brew 前缀 /opt/homebrew（Intel 老机器是 /usr/local）
- Node 用 fnm 管，全局只装 pnpm，禁止 npm -g
## 运行时
- Python 走 ~/dev/.venvs/<project>，不碰系统 Python
## 网络
- 代理 http://127.0.0.1:7890，拉境外依赖前先 export
## 禁止事项
- 不改 /etc/hosts；不全局 pip install
```

执行细节三点：

- 每条命令必须是你真实跑通过、确认过输出的，不是从文档抄的；行尾可标验证日期，把条目当带 TTL 的缓存。
- 环境差异用"每机一份"解决，别在一个文件里写 if-else。多机同步 workspace 时，机器特定的 TOOLS.md 走独立分支或不入库，共享的只有模板。
- 维护靠反馈闭环：agent 一碰到环境报错（路径不存在、命令缺失），修完当场更新文件。我还会定期让它自审——"逐条运行检查命令，报告与文件不符的项"——人工复核后再改。

## 踩坑点

- **绝对路径跨机会断**：workspace 走 git 同步时，`/Users/xxx` 在 Linux 上就是废的，用 `~` 或在各机文件里分别写清。
- **和 AGENTS.md 混职责**：前者管行为规范（怎么做事），后者管环境事实（机器长什么样）。混写两边都会被稀释。
- **文件贪长**：超过两百行，关键条目会被淹没，模型对中后段内容的遵循明显下降。冷门细节宁可让 agent 现场探测，也不要预先堆进去。
- **别假设写了就一定被遵守**：禁止事项要在 AGENTS.md 里再强调一遍，重要的事说两遍。

## 可复用建议

- 新机初始化时让 agent 自己起草：跑 `uname -a`、`which` 和各运行时版本探测，输出初稿，你逐条核实。比手写快且不易漏。
- 把它当配置代码对待：改动进 git，commit message 写清原因（如"升级 arm64，brew 前缀变更"）。
- 每次系统大版本升级后做一次审计，连续两次未命中的条目直接删。

## 总结

TOOLS.md 不是写给人的文档，而是你和 agent 之间关于这台机器的契约：写小、写实、一机一份、不放密钥、持续对账。做到这五点，它才不会沦为又一个没人维护的 README，而是 agent 在你机器上可靠干活的地基。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/db0ffb42ba332379.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/1881d0775da616b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/a764ecf1749b4369.png)

