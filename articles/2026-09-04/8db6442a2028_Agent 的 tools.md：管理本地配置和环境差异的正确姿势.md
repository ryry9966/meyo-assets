---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36063
source: 综合讨论
publishedAt: 2026-09-04
---

# 背景

Agent 跑久了会发现一个现实问题：真正干活的不只是模型，还有它本地的工具链——shell 脚本、MCP server、CLI 二进制、各类插件。这些工具在每台机器上的状态都不一样：笔记本上是 Homebrew 装的，服务器上是 apt，容器里可能是自己编译的。配置同步靠 git 很方便，但路径和环境没法同步。

常见的做法是把工具说明写进 system prompt，或者散落在各处配置里。结果就是：换一台机器，agent 满嘴跑工具，一调用就 `command not found`，白白烧掉好几轮。

# 问题

归纳下来三类：

1. **路径漂移**：同一工具在不同机器路径不同，硬编码必炸。
2. **能力误报**：prompt 里写了某个工具，实际本机没装，agent 反复尝试。
3. **信息重复**：MCP 配置、插件清单、system prompt 各说一遍，改一处忘三处。

# 做法

把 tools.md 当成「这台机器的工具事实清单」，核心原则：**机器本地生成，仓库只存模板**。

**1. 定一个最小结构。** 每个工具一条：名称、路径（用 `$HOME` 等变量，不写死）、用途、前置条件、不可用时的降级动作。

```markdown
## ffmpeg
- 路径: /opt/homebrew/bin/ffmpeg  （由探测脚本写入）
- 用途: 音视频转码
- 不可用时: 报告用户并建议安装，不要自行找替代品
```

**2. 写探测脚本 probe.sh。** 检查哪些二进制存在、版本多少、关键环境变量是否设置，渲染成 tools.md。装新工具后跑一次，或挂到安装流程末尾。

**3. 仓库只提交 tools.md.example 和 probe.sh**，真正的 tools.md 进 `.gitignore`。git 同步的是「如何描述」，机器各自持有「事实」。

**4. agent 启动时读一次 tools.md**（bootstrap 注入，或提供专门的读取工具），后续调用以它为准，不再依赖 prompt 里写死的假设。

**5. 加生成时间戳。** 排查问题时一眼看出是不是旧清单。

# 踩坑点

- **别把密钥写进去。** tools.md 会被读进上下文，也可能进日志。只写变量名引用，不写值。
- **别写太长。** 每个会话都要读，几百行封顶。描述「能力」，不要罗列全部参数。
- **别和 MCP/插件配置重复。** 写一句「docker 能力由 mcp-docker 提供」即可，参数留在原有系统里。
- **别忘了重新生成。** 最常见的翻车：装了工具没更新清单，agent 一直认为不可用。可以把 probe 挂进 shell hook 或安装脚本。
- **跨平台注意路径分隔符和 `~` 展开**，在探测脚本里统一处理，别指望 agent 自己纠正。

# 可复用建议

- 所有机器用**同一 schema**，prompt 不用分支，agent 行为才一致。
- 每个条目都写降级动作：「不可用就报告，不要猜」。
- 用 `diff` 对比两台机器生成的 tools.md，是发现环境差异最快的方式。
- 需要自动化校验时，让 probe 顺便输出一个 JSON sidecar，供 CI 或看门狗消费。

# 总结

tools.md 本质上是机器现实和 agent 假设之间的一纸契约。记住三个关键词：**生成而非手写**（probe 脚本是唯一事实来源）、**引用而非重复**（具体配置留在原有系统）、**校验而非信任**（时间戳加存在性检查）。做到这三点，同一份 agent 配置就能在笔记本、服务器和容器之间干净地跑起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/ac4e1f7e65bff98f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/e566789f9b432436.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/5b35a49254b8479b.png)

