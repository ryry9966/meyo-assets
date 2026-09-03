---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35990
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

Agent 能真正执行命令之后，最大的不确定性就从"任务怎么理解"转移到了"环境长什么样"。OpenClaw 的 workspace 里有一组约定文件：AGENTS.md 管行为约束，TOOLS.md 管工具怎么用。但不少人的 tools.md 要么空着，要么写成了工具介绍散文，没发挥它真正的作用——给执行者一份"这台机器的环境事实清单"。

## 问题

没有这份清单时，典型故障长这样：

- 项目锁定了 pnpm，Agent 按通用经验跑 `npm install`，装出两套 node_modules；
- 本机只有 `python3`，Agent 反复试 `python`，一个会话烧掉几千 token；
- 换台机器跑同一个 workspace，端口、路径、MCP server 全对不上，错误集中爆发；
- 团队环境各异，机器相关约定要么污染仓库配置，要么靠口头传承。

本质是环境事实没有单一出处，Agent 只能靠猜，猜错就进入"执行—报错—修正"循环，而且每个会话都要重猜一遍。

## 做法

**1. 分层：base + local。** `tools.md` 提交进仓库，写团队共识："统一 pnpm""测试命令是 `just test`"。`tools.local.md` 加入 .gitignore，写本机事实：绝对路径、端口占用、本地 MCP server 启动方式。在 AGENTS.md 里写明读取顺序：local 覆盖 base。

**2. 只写探测不到的事实。** package.json、Makefile、.env.example 已经是事实来源，不要复读。写 Agent 难以自动发现的：shell 别名、需要先 source 的脚本、代理设置、非标准安装位置。

**3. 每行一条可验证的事实。** 用"键 + 值 + 验证命令"的格式，别写叙事段落：

```
package_manager: pnpm        # pnpm --version
python: /usr/bin/python3     # 3.11
mcp_local: git, sqlite       # 启动脚本见 ~/.config/
```

**4. 能校验、能再生。** 写个十几行的脚本 dump 环境事实，生成 local 版草稿；再配一个 check 模式逐条跑验证命令，失败即提示环境漂移。新机器初始化 = 跑一次脚本 + 人工补几行。

**5. 控制体积。** 这份文件每次会话都进上下文，给自己定预算：超过 80 行就该删减合并。

## 踩坑点

- **写进敏感信息。** tools.md 会整体进入模型上下文。密钥、token、内网地址只写"去哪找"，不写值本身；local 版即使 gitignore 了也可能被截屏或复制出去。
- **写成散文。** 三段式"环境介绍"两周必过期，只留事实，不留叙事。
- **与已有文件重复。** 同一事实出现两处必然漂移，Agent 读到冲突版本比没有更糟。
- **漏掉 gitignore。** local 版带着内网路径进了仓库，撤都撤不干净。
- **以为 Agent 会记住。** 会话之间它什么都不会记住，文件就是它的记忆；改了环境不改文件，等于没改。

## 可复用建议

- base 版变更走 code review：团队约定先改 tools.md，再改人脑子；
- 校验脚本挂进 shell 初始化或 CI 自检，漂移早暴露；
- 模板里留一个"已知坑"小节，专收"只在某台机器出现"的怪问题；
- 这套分层同样适用于 MCP/插件配置：共享清单进 base，个人凭据路径进 local。

## 总结

tools.md 解决的不是技术难题，而是事实归属问题：环境事实应该有明确、分层、可校验的出处，而不是散落在各人脑子里等 Agent 每次去猜。搭一个初版只要一个下午，收益是每次会话少烧一轮试错 token。今天就从记录你机器上最容易猜错的三件事开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/dbf20c7ffa7accb4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/2c49bed082102162.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/dc1f253c30dcc64c.png)

