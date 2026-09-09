---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 36789
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

跑 Agent 这半年，最烦的不是模型不行，而是"同一套指令换台机器就不对"。我的常驻环境有三个：macOS 开发机、一台 Linux 小主机、CI 容器。三者差异很实际：brew 前缀分别是 `/opt/homebrew` 和 `/usr/local`；CI 里没有 docker daemon；BSD sed 和 GNU sed 参数不兼容。

Agent 不了解这些差异就只能猜。猜对是运气，猜错就是一串失败的工具调用。

## 问题

环境信息目前散落在三处，互不同步：

- shell rc 里的 alias 和 PATH；
- MCP 配置里声明的 server 列表；
- 系统提示词里手写的"本机情况说明"。

结果是：提示词很快过期；换机器或来新同事，环境靠口口相传；Agent 经常调用根本没装的工具，或用了 CI 里跑不通的命令。

## 做法

核心思路：把"这台机器上 Agent 能用什么、有什么坑"收敛成仓库里的一份 `tools.md`，版本化管理，Agent 启动时读取。步骤：

**1. 盘点。** 在每个环境跑一遍 `command -v` 和 `--version`，记录实际存在的工具、路径、版本，不凭记忆写。

**2. 写文件。** 模板大致如下：

```markdown
# tools.md —— 本机工具清单（禁止放密钥）
## 基础环境
- os: macOS 14 / arm64
- node: v20.11（fnm 管理）
## 路径与平台差异
- brew 前缀 /opt/homebrew；Linux 下为 /usr/local
- 无 GNU sed，文本替换用 gsed
## MCP
- filesystem、fetch 已在 mcp.json 声明，此处不重复枚举
## 禁止事项
- CI 禁止 docker compose up（无守护进程）
```

**3. 接入。** 在系统指令里加一行指向该文件，并确认你的 harness 是会话启动时读一次，还是每轮注入——这决定文件能写多长。

**4. 校验。** 写个十几行的脚本，比对 tools.md 声明与实际环境，本地和 CI 都跑，防过期。

**5. 维护。** 工具变更和代码变更走同一个 PR，评审时把它当代码看。

## 踩坑点

- **密钥入库。** 只写变量名和用途，值留在 `.env`，这是红线。
- **越写越长。** 超过两百行就是在烧上下文。用分层解决：全局一份放 `~/.config`，项目一份覆盖差异。
- **过期比缺失更糟。** Agent 对写下来的内容是信任的，一个未更新的版本号会引发连锁错误，drift-check 必须真跑。
- **两份真相。** MCP 列表已动态声明，tools.md 再枚举一遍迟早打架，只引用不复制。
- **平台差异写得太含糊。** "注意 sed 差异"没用，要写到"用 gsed 替代"的粒度。

## 可复用建议

- 完成标准定成一句话：**新同事照着仓库和 tools.md，十分钟能在新机器上复现可跑的环境**。做不到就是文件没写完。
- 全局 + 项目两级覆盖，与 dotfiles 思路一致，别发明新机制。
- 每次因"环境不同"踩坑，修完顺手补进 tools.md 对应章节——这是它保持准确的唯一方式。

## 总结

tools.md 的本质是把环境差异从隐性知识变成显性契约：人可读、可评审、可校验。它不解决工具管理本身，解决的是 Agent 与真实环境之间的信息不对称。成本低、收益稳定，值得作为每个 Agent 项目的标准文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/6c1c81bf3a960f4c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/df3023002df54293.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/55294c3dda769792.png)

