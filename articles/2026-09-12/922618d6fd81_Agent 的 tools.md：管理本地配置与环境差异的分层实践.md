---
title: Agent 的 tools.md：管理本地配置与环境差异的分层实践
feedId: 37229
source: 综合讨论
publishedAt: 2026-09-12
---

# 背景

跑 Agent 的人多半遇到过这种场景：同一句指令，在你机器上跑得好好的，换台机器或换个同事，Agent 开始用错包管理器、找错路径、调错版本的编译器。原因不神秘——Agent 的能力来自模型，但它的“环境认知”完全依赖你喂给它的上下文。这份上下文里最容易缺的，就是本地工具链的事实。

tools.md 的思路很简单：把 Agent 需要知道的环境事实写成一个显式文件，让它启动时读，而不是每次在 prompt 里临时拼。

# 问题

不写下来的代价是具体的：

- **命令漂移**：仓库已迁到 pnpm，Agent 仍在跑 `yarn install`；
- **路径假设**：文档里写死的绝对路径只在某一台机器存在；
- **平台差异**：`sed -i` 在 macOS 和 GNU 上行为不同，Agent 按它“记得”的写法改坏了文件；
- **环境不可复现**：三个人三个环境，同一个任务三种结果，排障无从对齐；
- **安全事故**：为了省事，把 key 直接贴进全局 system prompt。

根因只有一个：环境知识是隐性的，散在 shell rc、README 和口口相传里。

# 做法

核心是**分层 + 固定字段 + 校验**。

**1. 分三层：**

- 全局层 `~/.agent/tools.md`：机器级事实。OS、架构、Python 由 pyenv 管理、默认代理、docker 是否可用。不入库。
- 仓库层 `tools.md`：项目约定。包管理器、lint/test 命令、端口、compose 用法。入库，走 PR。
- 本地层 `tools.local.md`：个人覆盖，写本机路径、端口冲突的临时方案。gitignore。

**2. 每个条目用统一字段**：名称 / 何时用 / 命令 / 版本约束 / 已知坑。尤其是“已知坑”，比正确用法更值钱。

**3. 接入 Agent**：在项目级指令里明确写“启动后先读 tools.md 和 tools.local.md，冲突时仓库层优先”。MCP 场景可以在 server 初始化时注入路径。启动日志里确认它真的读了。

**4. 防腐烂**：配一个 `tools doctor` 脚本，逐条检查命令是否存在、版本是否达标，CI 里跑一次，文档坏了当场知道。

一个最小条目示例：

```markdown
## pnpm
- 何时用：任何依赖安装/脚本执行
- 命令：pnpm install / pnpm test
- 约束：>=9，勿用 npm 或 yarn
- 已知坑：node 由 volta 管理，直接装全局 node 会覆盖
```

# 踩坑点

1. **写了 secret**。tools.md 只写“从哪个环境变量读”，永远不写值本身。
2. **写得太长**。超过一两百行就该拆，或只留索引让 Agent 按需查。上下文是预算，不是仓库。
3. **优先级没声明**。全局层和仓库层冲突时 Agent 会随机站队，必须在文件里写清覆盖关系。
4. **只写理想路径**。Agent 最常栽在例外情况上，把“macOS 的 `sed -i` 要加备份后缀”这类条目写进去，收益最大。
5. **以为写了就有人读**。加载时机要显式，验证要留痕，别靠猜。

# 可复用建议

- 把 tools.md 当代码：改命令必须过 PR，review 时顺带确认 doctor 脚本还能过。
- 新人 onboarding 直接复用：让新同事的 Agent 先读 tools.md 再干活，等于对文档做实时测试。
- doctor 脚本从简单开始：`command -v` + 版本比对，五十行 shell 够用，别一上来搞框架。
- 层级宁少勿多。三层是经验上限，再加“团队层”基本没人维护。

# 总结

tools.md 不是什么新协议，就是把“环境即代码”落到 Agent 工作流里：事实显式化、层级清晰、可校验。它不会让 Agent 变聪明，但能砍掉一大类最烦人的低级故障——那些你花一小时排查、最后发现只是“它不知道你机器长什么样”的问题。先从仓库层一个十行的文件开始，比争论规范有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f7018026486ecc54.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/591427a549d16b2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/98853d52c53c654f.png)

