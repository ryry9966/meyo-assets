---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35772
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

OpenClaw 这类本地 Agent 的特别之处在于：它不只对话，还会在你的机器上执行命令、改文件、调服务。它对这台机器的认知，几乎全部来自 workspace 里的 markdown 文件——AGENTS.md 约定行为，tools.md 承载“这台机器怎么用”。

只有一台机器时，你感觉不到 tools.md 的价值。一旦 workspace 在笔记本、家里主机和 VPS 之间同步，或者配置被同事拿去复用，环境差异就开始咬人。

## 问题

几个典型症状：

- Agent 在 Linux VPS 上执行 `brew install`，因为 tools.md 是从 MacBook 同步过来的；
- 路径写死，`~/projects/xxx` 在另一台机器上根本不存在；
- 需要 sudo 的操作，Agent 硬跑，然后卡住或静默失败；
- 同一份 tools.md 多机互相覆盖，谁最后同步谁正确；
- 更隐蔽的：token 和密码混进了 tools.md，随 git 推到了远端。

根因只有一个：tools.md 被当成了随手记事本，而不是这台机器的接口文档。

## 做法

我的做法是把它拆成两层，按三步落地。

**第一步：拆层。** `tools.md` 只放跨机器成立的共享约定（封装脚本、命名规范、日志习惯）；环境差异全部放进 `tools.local.md`，并加入 `.gitignore`，每台机器一份，不进版本库。

**第二步：定模板。** tools.local.md 保持最小结构：

```markdown
# <机器名> — 2025-01 更新
## 系统
Ubuntu 24.04 / apt / 无 GPU / 时区 Asia/Shanghai
## 关键路径
- 工作区: /home/user/workspace
- 日志: /var/log/openclaw/
## 常用操作（已验证）
- 重启网关: systemctl --user restart openclaw
- 实时日志: journalctl --user -u openclaw -f
## 禁区
- 不要用 sudo
- 不要直接改 nginx 配置，先在 staging 验证
```

**第三步：守原则。** 只写验证过的命令，没跑通的不写；写成可直接复制执行的命令，而不是描述性语句；文件开头加一句“环境信息可能过期，执行前先核实”，让 Agent 优先用命令确认现状，而不是盲信文档。

## 踩坑点

1. **和 AGENTS.md 打架。** 两边都写“怎么部署”且内容不一致，Agent 行为会随机化。定好边界：AGENTS.md 管行为准则，tools.md 管环境事实。
2. **文件越长越没用。** 超出一屏，Agent 检索时容易被无关内容带偏，定期删除失效条目。
3. **过期信息比没有信息更危险。** 系统升级后忘了更新，Agent 会拿着旧路径反复失败。给每个 section 加更新日期，存疑条目主动核实。
4. **本地文件忘了 gitignore。** tools.local.md 一旦推上远端，机器名、路径、内网地址全部泄露。建仓库第一天就配好。

## 可复用建议

- 新机器初始化时，先让 Agent 扫一遍环境（OS、包管理器、已装服务），自动生成 tools.local.md 草稿，人工核对后启用，比手写快且不易漏。
- 把 tools.md 当基础设施代码对待：小、准、有版本、分环境。
- 团队场景下，共享层进仓库走 review，机器层各自维护，互不污染。

## 总结

tools.md 的本质，是给 Agent 的一份环境契约。写得好，Agent 在哪台机器上都像本地人；写得差，它就是一个拿着过时地图乱指路的向导。两层拆分、只写验证过的命令、定期修剪——这三件事做到位，环境差异就从玄学变成了普通配置项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/d6271a1606869804.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/44e673eb6a834e97.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/2a4fe35b0d5c6a54.png)

