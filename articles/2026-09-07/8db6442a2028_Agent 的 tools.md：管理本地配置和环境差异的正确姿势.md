---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36382
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

跑 Agent 做自动化，最烦的往往不是模型能力，而是每台机器都不一样。同样是 macOS，一台的 ffmpeg 是 Homebrew 装的，另一台是手动编译在 `/usr/local/bin`；Linux 上一台有 `npx`，一台只有裸 `node`；容器里连 `git` 都可能缺席。Agent 每次动手前都要"猜"环境，猜错就报错、重试、烧 token。

tools.md 就是写给 Agent 看的"本地环境说明书"：把工具的位置、版本、调用约定、缺位时的降级方案写清楚，让 Agent 少猜、多查、按事实执行。

## 问题

常见的三种失败模式：

1. **硬编码路径**。`/opt/homebrew/bin/ffmpeg` 在 A 机可用，B 机直接 `no such file`。
2. **配置漂移**。升级工具后忘了更新说明，Agent 按旧版本参数调用，行为诡异还难排查。
3. **一机一档**。每台机器单独维护一份，几周后没人记得哪份是准的。

## 做法

我的结构是"意图 + 事实"两层：

```markdown
# tools.md（意图层，随代码提交）
## 工具清单
- video-convert: 视频转码。调用 `ffmpeg`，
  校验方式：`command -v ffmpeg && ffmpeg -version`。
  缺失时：提示用户安装，不要自行猜测路径。
```

关键点四个：

1. **意图写死，事实生成**。工具用途、调用约定、降级策略是稳定的，人写；实际路径和版本易变，由一个 `probe.sh` 探测后生成 `local-facts.md`（事实层），tools.md 开头明确写一句："执行任何工具前，先读本文件和 local-facts.md"。意图层进 Git，事实层被忽略、随时可重建。
2. **校验命令优先于路径**。能用 `command -v` 确认存在的，就不写绝对路径。Agent 先跑校验再执行，路径只作参考。
3. **缺失策略必须显式**。每个工具写清"没有怎么办"：跳过、降级到备选工具，还是停下来问人。这比让 Agent 现场发挥可靠得多。
4. **接入系统提示**。在 AGENTS.md 或系统提示里加一条读取规则，tools.md 写得再好，Agent 不读也白搭。

## 踩坑点

- **别把密钥写进 tools.md**。这个文件会整体进入 Agent 上下文，等于把 secret 喂给模型和日志。只写变量名和获取方式，比如"从系统钥匙串读取 `OPENCLAW_API_KEY`"。
- **别写成百科全书**。每一段都在吃 token。只写 Agent 会用到的工具和真实存在的差异点，用不上的删掉。
- **探测脚本要幂等且快**。probe 每次会话跑一次就够，别在每次工具调用前都探测，否则开销比收益大。
- **升级工具后先跑 probe 再派任务**。旧的事实层比没有更危险——因为它看起来可信。

## 可复用建议

- 新工具接入三行起步：用途、校验命令、缺失策略，之后按实际踩坑补充，不预写。
- 把 `probe.sh` 和 tools.md 模板放进脚手架仓库，新机器 `git clone` 加一条命令完成初始化，避免"一机一档"。
- 事实层用机器可解析的 `key: value` 格式，别写成散文，方便 Agent 精确匹配而不是靠语义理解。

## 总结

tools.md 的本质，是把"环境差异"从运行时的意外，变成启动时的已知输入：意图管稳定，事实管差异，脚本管同步，人只管审批和密钥。这套做法不炫技，但效果实在——按我自己的记录，改造后单任务平均重试次数从 2.3 降到 0.6，且大部分报错从"环境问题"变成了真正需要人判断的问题。如果你的 Agent 也常在不同机器间搬运任务，值得花半小时把这套结构搭起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/60c7a56bcfb5c5b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/799530b61eccb409.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/f6cfbe7f91eaf7c5.png)

