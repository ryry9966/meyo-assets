---
title: Agent 的 tools.md：共享层 + 本地生成，管住环境差异
feedId: 37096
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

跑 OpenClaw、MCP 或各类自动化任务的人大概率遇到过这个场景：同一套指令，在自己的 Mac 上跑通，换到 Linux 服务器或容器里就翻车。原因往往不是代码，而是环境差异——路径不同、工具缺失、shell 不一样、Python 版本对不上。

我们的做法是给每个 agent 工作区放一份 `tools.md`，把它当成 agent 认识当前环境的"接口文档"，而不是给人看的 wiki。运行一段时间后总结出了一套还算稳的管理方式，分享出来。

## 问题

此前踩过的坑：

- agent 凭训练先验猜命令，本机没装对应工具时就编造参数
- 配置散落在 `.bashrc`、`.env`、内部 wiki 三处，互相矛盾
- 换一台机器，文档立刻过期一半
- 有人把 token 直接写进文档还提交进了仓库

核心矛盾在于：agent 需要确定性的工具信息，但环境本身是易变的。手写文档追不上环境变化。

## 做法

**1. 分层：共享 + 本地**

- `tools.md`：机器无关的部分，进仓库，团队共享
- `tools.local.md`：机器相关部分（绝对路径、端口、GPU 有无），加入 gitignore，由脚本生成
- 约定优先级：local 覆盖 shared，agent 两份都读

**2. 写探测脚本，不手写环境段**

写一个 `probe.sh`，检测 OS、shell、docker、python、GPU 等，输出 markdown 片段填入 `tools.local.md`。换机器时重跑一次即可，杜绝手抄。

**3. 每个工具条目固定四行结构**

```markdown
### ffmpeg
- 用途：音视频转码、抽帧
- 调用：ffmpeg -i {in} ...
- 自检：ffmpeg -version | head -1
- 失败时：提示 brew install ffmpeg 或 apt install ffmpeg
```

自检命令很关键：agent 调用前先跑一遍，避免拿着过期信息干活。

**4. 控制体积**

一行用途加精确命令即可，细节外链到普通文档。tools.md 是占上下文的，不是技术分享。

## 踩坑点

- **别放密钥**：只写环境变量名（如 `OPENAI_API_KEY`），值放 `.env` 或系统钥匙串
- **别写绝对路径**：由探测脚本注入，或统一用 `~` 和环境变量
- **版本漂移**：文档里写"docker ≥ 24"，自检命令里就真的去比版本，否则等于没写
- **两份文件冲突**：我们遇到过 local 残留旧端口排查了一小时。规则写死：探测脚本全量重写 local 文件，不做增量合并
- **别放任 agent 自己维护文档**：让它改可以，但提交前必须过校验脚本，防止它"顺手美化"掉关键约束

## 可复用建议

- 起步模板三段就够：全局约定、工具条目、本机差异（引用 local 文件）
- 把"跑 probe + 校验 tools.md"挂进 devcontainer 的 post-create 和 CI，环境变更自动更新
- 团队评审只看共享层，local 层永远不进 code review
- 条目超过 30 个就按场景拆文件、按需加载，别让一份巨型文档吃光上下文

## 总结

tools.md 的本质，是把环境知识从口头约定变成可生成、可校验的配置即代码：共享层保证团队一致性，本地生成层消化机器差异，自检命令让 agent 在运行时自我纠错。我们的体会是——给 agent 的文档，越像接口定义、越不像散文，越靠谱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/8230062b08cf848c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/0050ac7f7e27c9d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/c2a47ad281b257da.png)

