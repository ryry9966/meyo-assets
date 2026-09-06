---
title: Agent 的 tools.md：本地配置与环境差异的分层管理
feedId: 36393
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

跑在本地机器上的 Agent 默认对你的环境一无所知：Python 是 conda 还是系统自带、容器用 docker 还是 podman、代理走哪个端口、哪些目录能动哪些不能动。这些信息只在对话里口头补充，换个会话就丢；写进全局系统提示词，又会在所有机器上生效。OpenClaw 的 workspace 提供了 TOOLS.md 这一层：每次会话注入上下文，相当于给 Agent 一份“本机说明书”。文件本身不复杂，问题出在管理方式上。

## 常见问题

- 环境事实散落在聊天记录里，每次重讲一遍；
- 一个 tools.md 多机同步，结果在 Linux 服务器上执行了 macOS 的 brew 命令；
- 把 API key、内网地址顺手写进去，随上下文进入日志；
- 文档与真实环境漂移，Agent 按过期说明执行，失败后还得靠人兜底。

## 做法

1. **分层**。全局一份 tools.md，只写跨机器稳定的约定（工作流、禁令、通用习惯）；机器差异单独放 tools.local.md 并加入 .gitignore，或按主机命名如 tools.homelab.md，在对应实例的 workspace 里覆盖。优先级规则要写死：local 覆盖 global，不给 Agent 自行仲裁的空间。
2. **结构化分节**。运行时与包管理 / 网络与代理 / 常用命令 / 禁止事项 / 已知坑，每节只放“事实 + 可直接执行的命令”：

```markdown
## 运行时
- node 由 nvm 管理；验证：`node -v && which node`
- python 走 pyenv；项目内执行 `python3.12 -m venv .venv`

## 禁止事项
- 不改动 .env 与 ~/data
- 不执行 sudo apt upgrade
```

3. **每条事实配一条验证命令**。Agent 执行前可以自查，验证失败即说明文档过期，这是对抗漂移的核心机制。
4. **写精确命令，不写描述**。“用新一点的 Python”是猜谜，`python3.12 -m venv .venv` 才是指令。
5. **变更即回写**。某条命令失败一次，当场修文档。tools.md 的可信度完全靠这条纪律维持。

## 踩坑点

- **机密进文档**：tools.md 会整体进入上下文，密钥和 token 一律不写，用环境变量引用；
- **写“期望”不写“现状”**：文档里的每条路径和命令都要真实跑过一遍再入库；
- **越写越长**：超过一两百行，关键信息会被稀释，定期删陈旧条目，宁可短而准；
- **双份文件冲突**：全局文件里写了 A、主机文件里写了 B，Agent 会随机采信，务必在文件头部声明覆盖关系。

## 可复用建议

- 全局文件进 git，环境变更时走 diff review，谁改了什么有据可查；
- 每月或大版本升级后，让 Agent 拿着 tools.md 逐条跑验证命令，做一次“审计会话”，输出过期条目清单；
- 换新机器时，先让 Agent 做环境探测、生成 tools.local.md 初稿，再人工修订，比手写快且不易漏；
- 禁止事项单独成节并保持精简，这一节是安全边界，不是备忘录。

## 总结

tools.md 的价值不在“多了一份文档”，而在把环境知识从易丢失的对话记忆，变成可版本化、可验证、可分层的资产。全局约定与机器差异分离、每条事实可自证、变更即时回写——做到这三点，Agent 在你每台机器上都能少猜、少错、少问。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c549a84dfb74b6d2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/896c4915601839e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/36d0a62463532a0b.png)

