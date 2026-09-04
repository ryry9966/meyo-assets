---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36126
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 的 agent 默认带着 exec 和文件读写工具，能跑命令、能改文件。于是每个认真部署过的人都会问同一个问题：模型一旦抽风，或者读到一段带提示注入的网页内容，会不会一条 `rm -rf` 把家目录清掉？

答案取决于你的部署姿势。OpenClaw 的 sandbox 设计前提不是"相信模型不犯错"，而是假设它一定会犯错，然后用多层围栏兜底。这套模型不复杂，值得每一层拆开看。

## 围栏是怎么叠起来的

**第一层：低权限专用用户。** Gateway 和所有工具进程跑在一个专用系统用户下，不是你的登录用户，更不是 root。它对你的 home 目录、系统配置天然没有写权限。这是最硬的一层：即使模型真的发出了 `rm -rf ~`，在内核层面就不成立。

**第二层：workspace 路径围栏。** 文件类工具执行前会把目标路径做 realpath 规范化，解析掉 `..`、相对路径和 symlink，然后校验是否落在 workspace 根内。workspace 里一个指向外部的软链接，解析后同样会被拒绝——因为校验的是解析后的真实路径，不是模型给的字符串。

**第三层：exec 沙箱。** exec 的 cwd 被钉在 workspace 内；开了容器模式时，命令直接在容器里执行，只 bind-mount workspace，非 root，drop 掉多余 capabilities。工作区之外的世界对它基本不可见。

**第四层：危险操作策略门。** 命令模式匹配到 `rm -rf`、`dd`、`mkfs` 这类高危片段，命中即转人工确认或直接拒绝，按你配置的策略走。MCP 工具同理：默认拒绝，逐个显式启用。

**第五层：审计日志。** 每条 exec 记录 cwd、命令、退出码。真出事，能完整复盘到哪一步穿了多少层。

## 踩坑点

- **用 root 跑 gateway**：所有围栏直接归零。为了省 sudo 密码这么干，等于亲手拆掉第一层。
- **容器挂了整个 /home**：只该挂 workspace。挂宽了，第二、三层同时失效。
- **字符串前缀校验**：自己写插件时如果只做 `path.startswith(workspace)` 而不 realpath，一个 symlink 就穿透。校验永远在工具层做，不要信模型自己声明"我确认在 workspace 内"。
- **MCP 旁路**：某个 MCP server 自带 shell 能力却没接入同一套策略，是最常见的漏点。每启用一个工具，先过一遍它能触达什么。
- **别指望提示词当防线**：网页、邮件里的注入内容诱导执行命令是真实场景，但真正的防线是上面五层，不是 system prompt 里那句"请小心"。

## 可复用建议

1. 默认最小权限，需要时显式开洞，用完收回。
2. 任何路径校验都先 realpath 再比对前缀，双向处理 symlink。
3. 破坏性命令一律过确认门，宁可多按一次确认。
4. 上线前做一次破坏性演练：明确让 agent 尝试删除 workspace 外的文件，验证它确实失败并留下日志。
5. 把演练固化成脚本放进 CI，升级 OpenClaw 或改完配置后重跑。

## 总结

Agent 不会误删文件，不是因为模型足够聪明，而是因为它即便发出了那条命令，内核权限、路径围栏、策略门会依次把它拦下。这是典型的纵深防御：任何一层被绕过，下一层还在。把模型当成不可信的输入方来设计权限，才是在生产环境里敢让它跑起来的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/6d6c26640efda0ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/385bd332db15ac1c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/96c584db7a51c04b.png)

