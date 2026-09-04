---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 36052
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

用 Agent 做本地自动化，工具层永远是最脆的一环。MCP server 声明了能力，OpenClaw 插件注册了命令，但真正执行时面对的是你这台机器：ffmpeg 在不在 PATH、Python 是 venv 还是 conda、代理端口是 7890 还是 1080。这些事实工具本身不会告诉你，Agent 只能猜。

我们的做法是把"这台机器上工具的真实状态"沉淀成一份 tools.md，作为 Agent 的启动上下文之一。它不是文档，是运行时事实的快照。

## 问题

早期把 tools.md 直接提交进仓库，很快遇到三个麻烦：

1. 同一份文件在 mac、WSL、CI 容器里表现完全不同。路径写死 `/usr/local/bin/ffmpeg`，换台机器 Agent 就开始瞎猜命令。
2. 有人顺手把 key、内网地址写进去，随仓库扩散。
3. 文件和实际安装的工具慢慢脱节，Agent 读到的是三个月前的描述，排查全在兜圈子。

根因只有一个：把"团队共识"和"机器事实"混在了一个文件里。

## 做法

拆成两层，职责分离：

- `tools.md`（进仓库）：只写团队层面的事实——依赖哪些工具、最低版本、用途、验证命令，不含任何机器相关路径。
- `tools.local.md`（gitignore）：每台机器各自的差异——实际可执行路径、版本、环境变量、代理设置、GPU 有无。

优先级明确：local 覆盖仓库版，Agent 启动时两份都读。

local 文件不手写，用探针脚本生成：`scripts/env_probe.sh` 检测 OS 和包管理器，`command -v` 逐个定位工具并读取版本，输出固定格式 markdown。换新机器跑一遍，30 秒出结果。

每个条目固定五字段，方便 Agent 稳定解析：

```markdown
### ffmpeg
- path: /opt/homebrew/bin/ffmpeg
- version: 6.1  # 要求 >= 6.0
- verify: ffmpeg -version
- notes: 硬件编码不可用，转码走 libx264
```

再提供一个 `toolcheck`：Agent 执行工具前先跑 verify，失败就停下报告，而不是带着过期假设继续执行。

## 踩坑点

- **密钥别写进任何 tools 文件**。Agent 上下文会被日志和 trace 记录，泄漏面比 `.env` 大得多，local 里只写变量名。
- **描述与实际注册的工具要同源**。tools.md 写了某插件但 MCP 配置没启用，Agent 会反复调用然后报错。我们的规则：生成脚本先和 MCP 配置核对，对不上就不输出该条目。
- **控制长度**。上下文预算有限，散文不如表格，每个工具 4–6 行封顶，边缘情况收进 notes 一行。
- **缓存生效时机**。多数框架只在会话启动时读一次上下文，改完不重启等于没改。踩过之后流程固定为：改配置 → 重启会话 → toolcheck。
- **跨平台路径**。WSL 下 Windows 与 Linux 工具混用，探针要按 shell 环境分别探测，别假设单一 PATH。

## 可复用建议

1. 五字段模板（name / path / version / verify / notes）团队通用，Agent 的解析逻辑写一次就够。
2. env_probe 脚本和 tools.md 一起进仓库，README 写明"新机器先跑 probe 再起 Agent"。
3. 把 verify 命令搬进 CI 做冒烟测试，仓库版的版本约束失效能第一时间暴露。
4. probe 输出带时间戳，local 文件超过 30 天提示重新生成。

## 总结

tools.md 的价值不在写文档，而在把环境差异变成 Agent 可消费的结构化事实。分层（共识与事实分离）、生成而非手写、验证先于执行——这三条撑住了多环境下的稳定性。文件本身很朴素，省下来的是无数次"为什么在我机器上是好的"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/cf942d9bab3e9b8a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/964fcbbe0a9f8ba5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/e6f7ae56ee48b8f1.png)

