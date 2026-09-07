---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36420
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 的 Agent 一旦开始真正干活，就绕不开本地环境：ffmpeg 装在哪、Python 用哪个 venv、浏览器驱动什么版本、MCP server 的启动参数是什么。这些信息如果只存在于你的脑子和某次调试用的 prompt 里，换台机器 agent 就会开始猜——猜错的结果就是幻觉路径、瞎执行、浪费 token。

我们的做法是把"工具清单"固化成一份 agent 可读的 `tools.md`，让它成为环境事实的唯一入口。

## 问题

常见的几种翻车方式：

- 绝对路径写死在 skill 或系统 prompt 里，换机器/换容器直接失效；
- 同事的工具在 `/opt`，你的是 Homebrew，agent 按 A 机器的描述去调 B 机器；
- 为省事把 API key 顺手写进配置然后提交了仓库；
- MCP 配置和 prompt 里的工具描述各自演化，慢慢对不上；
- Windows 路径带空格或中文、agent 拉起的 shell 和你交互 shell 的 PATH 不一致。

本质是同一个：**环境事实没有分层，也没有校验。**

## 做法与步骤

**1. 分层：base + local。**

- `tools.md`：提交进仓库，只写与机器无关的事实——工具是干什么的、大致怎么调用、前置条件；
- `tools.local.md`：进 `.gitignore`，只放本机覆盖项——绝对路径、版本、特有参数；
- 机密永远不落在这两个文件里，用环境变量，tools.md 里只写变量名。

```markdown
# tools.md（base 层）
transcode:
  what: 音视频转码
  cmd: ${FFMPEG_BIN:-ffmpeg}
  check: ffmpeg -version

# tools.local.md（本机覆盖）
transcode:
  cmd: /opt/homebrew/bin/ffmpeg
```

**2. 明确加载顺序。** 环境变量 > tools.local.md > tools.md，后者只补前者未定义的键。不做深合并，一层覆盖一层，行为可预测。

**3. 启动时校验，而不是祈祷。** 写一个十几行的 `doctor` 脚本：读合并结果，逐条执行 `check`，失败就打印哪台机器缺什么。会话启动前跑一次，失败直接退出。

**4. 把"不要猜"写进 tools.md 本身。** 第一行就告诉 agent：任何工具未通过校验时，报告缺失，禁止凭记忆编造路径。这句话比后面所有条目都值钱。

**5. 和 MCP 配置划清边界。** tools.md 描述语义（什么时候用哪个工具），MCP 配置持有连接参数。tools.md 里只放指向关系，不复制内容，避免两份真相漂移。

## 踩坑点

- **写成散文。** tools.md 每个会话都进上下文，长篇大论纯烧 token。条目化，一句 what、一句 how，够了。
- **校验只存在于文档里。** "理论上能跑"等于不能跑，`check` 必须在启动时真的执行。
- **编辑后不生效。** 有的运行时会缓存工具清单，改完记得重启会话或触发 reload。
- **覆盖层级太多。** 最多两层，再叠 env-specific、user-specific 就没人说得清最终值从哪来。
- **子进程 PATH 与交互 shell 不一致。** 关键工具在 tools.local.md 写绝对路径，别赌 PATH。

## 可复用建议

- 新机器初始化固化成一条命令：跑 doctor → 自动生成 tools.local.md 骨架 → 人工填空；
- tools.md 的改动走 PR，把它当代码而不是便签；
- schema 保持稳定（what / cmd / check 三件套），agent 对结构的记忆比对散文可靠得多。

## 总结

tools.md 不是高深机制，就是三件事：**分层（base + local）、校验（启动时 doctor）、不重复（MCP 归 MCP）**。做完之后最直观的变化：换机器从半天调试变成填五分钟空，agent 从"偶尔猜路径"变成"缺了就报告"。工程上便宜，收益上稳定，值得成为团队默认约定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/cb5afc2713403c9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/05ce9d57af415c60.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/421d0c6ee394a757.png)

