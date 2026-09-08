---
title: Agent 的 tools.md：用一份「环境说明书」管住本地配置差异
feedId: 36680
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

跑 Agent 的人大概都遇到过：同一套工具链，在自己的 Mac 上好好的，搬到 Linux 服务器上就找不到可执行文件；同事拉了仓库，Python 路径、浏览器内核位置、MCP 端点全对不上。Agent 的能力边界很大程度上由本地环境决定，但这份「环境事实」往往散落在 prompt、shell 脚本和大家的脑子里。

tools.md 的定位，就是把这些事实收敛成一份机器可读、人也看得懂的约定文件：Agent 启动时读取它，明确知道本机有哪些工具、怎么调用、有什么限制。

## 问题

几个常见反模式：

1. **环境信息硬编码**。绝对路径写死在 prompt 里，`/Users/xxx/...` 一换机器就失效。
2. **秘密混进配置**。API key 直接写在文档里提交到 git，泄漏只是时间问题。
3. **文档漂移**。工具升级后没更新 tools.md，Agent 按旧说明调用，报错还很难懂。
4. **平台差异不标注**。macOS 用 brew、Linux 用 apt 的事，文档里只写了一个。

## 做法

我们现在的实践分三层：

**第一层：tools.md 只写事实，不写秘密。** 每个工具一个条目，字段固定：

```markdown
## browser-control
- 用途：无头浏览器操作，截图与表单填写
- 调用：通过 MCP server `browser` 暴露
- 依赖：Chromium >= 120，路径见 `BROWSER_PATH`
- 平台差异：macOS 需先 `brew install`；Linux 依赖 `libnss3`
- 已知限制：不支持下载大于 100MB 的文件
```

**第二层：差异交给环境变量和本地覆盖文件。** 路径、端点一律用 `${VAR}` 占位，实际值放在 `tools.local.md`（进 `.gitignore`），或由 shell 环境 / `.env` 注入。仓库里的 tools.md 是骨架，本机的 tools.local.md 是血肉。

**第三层：启动前校验。** 写一个十几行的 doctor 脚本，逐条检查 tools.md 声明的依赖是否真实存在：二进制在不在 PATH、端口通不通、版本对不对。Agent 拿到的是校验过的环境，而不是文档承诺的环境。

## 踩坑点

- **别把 tools.md 当 prompt 写**。有一版写了大段「使用建议」，Agent 反而抓不住关键字段。条目化、字段固定，比文笔重要。
- **占位符要有默认值兜底**。`${BROWSER_PATH:-/usr/bin/chromium}` 这种写法能省掉一半「环境没配好」的求助。
- **变更要走和代码一样的流程**。tools.md 的改动进 PR，CI 里跑 doctor 脚本，否则漂移是必然的。
- **控制长度**。Agent 每次都会读它，冗余条目是真金白银的 token，整份文件建议几百行内。

## 可复用建议

- 把 tools.md 当「环境 API 文档」维护：有变更就更新，有疑问以它为准。
- 团队共享的是骨架（结构 + 公共工具），个体差异全部收敛到 local 覆盖层。
- doctor 脚本同时服务人和 Agent：新人入职跑一遍，Agent 启动也跑一遍，同一份真相。
- 定期删条目。下线的工具直接移除，比留着标「已废弃」更安全。

## 总结

tools.md 解决的不是高深问题，就是把「本机环境长什么样」从口口相传变成版本化的事实来源。原则只有三条：**文档写事实、差异进变量、启动先校验**。做到这三条，换机器、换同事、换 Agent 框架时，环境这块基本不会再是事故现场。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/4746609000317a79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/c67ad91604e7af50.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/5638d533aadd006d.png)

