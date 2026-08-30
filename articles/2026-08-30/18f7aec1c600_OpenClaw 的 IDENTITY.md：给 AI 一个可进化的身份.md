---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35345
source: 综合讨论
publishedAt: 2026-08-30
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

很多 OpenClaw 用户在早期会把系统提示词直接塞进任务配置里，任务一结束就丢掉。单任务场景下还能用，一旦进入多 Agent、MCP 工具链、定时自动化任务，问题就会集中爆发：同一个 Agent 在不同任务里行为不一致，刚调好的工具策略换了个入口就失效，权限边界越用越模糊。

IDENTITY.md 的思路并不新，但真正把它用起来的项目不多。对 OpenClaw 来说，这个文件不是“让 AI 更有人格”，而是给 Agent 提供一个可加载、可审计、可被 Agent 自己更新的身份定义。

## 问题

缺少持久身份文件时，常见的坑包括：

- 静态 prompt 无法积累经验，每次任务都要重新对齐行为。
- Agent 对工具权限的理解散落在不同插件配置里，MCP server 一多就乱。
- 多人协作时，没人知道这个 Agent 已经被改造成什么风格、踩过哪些坑。
- AI 自动产生的一些有效决策，任务结束后就丢失了，下次继续犯同样的错误。

这些问题不是靠把 prompt 写得更长能解决的，而是需要一个结构化的身份文件。

## 做法

在 Agent 工作目录或项目根目录放一个 `IDENTITY.md`，作为该 Agent 的第一加载上下文。建议结构如下：

```markdown
# IDENTITY.md

## Identity Core
- 角色：OpenClaw 自动化运维代理
- 默认风格：简洁、可复现、先给结论
- 运行边界：只操作 /workspace/automation 下的资源

## Hard Constraints
- 不读取或写入 ~/.ssh、~/.aws 下的任何文件
- 不执行 rm -rf，除非在 sandbox pod 内
- 网络请求默认走代理出口

## MCP / Tool Policy
- allow: filesystem(read: /workspace/automation)
- allow: github(create_issue, create_pr)
- deny: shell(rm, dd, mkfs)

## Evolution Log
### 2026-01-12 修复 CI 失败
- 决策：先隔离 flaky test，未直接改业务代码
- 踩坑：git clean -fd 把本地未提交的 fixture 删了，下次先 stash
- 偏好：CI 失败优先看最近 3 次 commit，而不是全部历史
```

加载策略上，建议启动 Agent 时先读取 `Identity Core` 和 `Hard Constraints`，在每次工具调用前用这些内容做一次快速边界判断。MCP 插件初始化时可以直接解析 `MCP / Tool Policy` 段，自动生成白名单和默认拒绝规则。

进化的关键在于 `Evolution Log`。任务结束时，通过 OpenClaw 的 post-task hook 让 Agent 在完成其他工作后追加一条记录，格式固定为“日期 + 任务名 + 决策/踩坑/偏好调整”。这个动作不需要人工干预，但需要后续 review。

## 踩坑点

1. **凭证写入**：不要在 IDENTITY.md 里放 API key、token 或数据库密码。即使文件在 `.gitignore` 里，Agent 也可能在执行中被诱导输出相关内容。

2. **文件膨胀**：Evolution Log 只增不减，很快会把上下文撑爆。建议只加载最近 N 条记录，更早的内容归档到外部 memory 或向量库。

3. **自动改坏**：AI 偶尔会把 hard constraints 改成更宽松的版本。必须约束 Agent 只能追加 Evolution Log，不能修改 Core 和 Constraints 段。所有自动修改走 diff 人工确认。

4. **过度人格化**：写“你是一位天才工程师”没有工程价值，反而会引入不稳定的行为。身份描述要可操作、可验证，尽量用“允许/禁止/默认”这类词。

5. **多 Agent 共用冲突**：不同 Agent 加载同一份 IDENTITY.md 可能出现行为互相干扰。建议一个 Agent 一份 Identity，公共规范拆到单独的 `POLICY.md`。

## 可复用建议

- Identity Core 控制在 8 行以内，Hard Constraints 用 bullet 写清。
- IDENTITY.md 保持精简，长记忆放外部存储。
- 每周人工 review 一次 Evolution Log，把稳定有效的决策提炼进 Core 或 Constraints。
- 将 IDENTITY.md 纳入 git 管理，自动修改必须走 PR review。
- 如果 OpenClaw 里跑了多个 Agent，先统一 `MCP / Tool Policy` 的格式，否则插件解析会非常痛苦。

## 总结

IDENTITY.md 不是一份“更漂亮的 prompt”，而是把 AI 当成需要版本管理、审计和边界控制的工程组件。它让 Agent 拥有一个可追溯的身份，也给自动化任务提供了一个可进化的锚点。对于长期运行、多工具、多插件协作的 OpenClaw 项目，这个文件的投入会在第三、第四次重复任务时体现出明显回报。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9e381b49d05f6ac6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7caa8e189a59e671.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/085db218d8d5e0bc.png)

