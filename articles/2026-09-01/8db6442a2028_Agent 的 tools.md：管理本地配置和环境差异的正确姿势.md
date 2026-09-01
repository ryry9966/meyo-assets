---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35676
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

跑 OpenClaw 的人很少只有一台机器：家里一台 Mac，公司一台 Linux 服务器，可能还有台 Windows 机器挂着自动化任务。Agent 的提示词是全局的，但每台机器的现实不一样——Python 版本、有无 GPU、代理端口、启用了哪些 MCP server、脚本放在哪个路径。这些差异处理不好，agent 在 A 机器上表现正常，换到 B 机器就开始编造不存在的命令。

tools.md 就是为此准备的：一份由你维护、给 agent 读的"本机事实清单"，放在 workspace 里，随会话进入上下文。

## 问题

三种常见的坏味道：

1. 路径、端口写死在系统提示词里，换机器就失效；
2. 工具说明散落在聊天记录和插件配置里，agent 只能靠猜；
3. 多台机器共用一份配置，谁也不敢改，最后文档和现实脱节——agent 照着文档调用，命令根本不存在。

## 做法

我把信息分三层：

- **全局提示词**：只写角色、规范、通用行为，不写任何机器相关事实；
- **tools.md**：写"这台机器有什么、怎么用"；
- **环境变量**：密钥只进 env，tools.md 里只写变量名和用途，绝不写值。

tools.md 的条目我固定一个骨架：

```markdown
## ffmpeg 转码
- 用途：压缩、抽帧
- 调用：ffmpeg -i {in} -vcodec libx264 {out}
- 自检：ffmpeg -version
- 备注：无 GPU，大文件加 -preset veryfast
```

关键是"自检"那行：agent 不确定环境时先跑自检命令，再决定是否调用，比把文档写得花哨管用得多。哪些 MCP server 在本机启用、各自能做什么，也照这个格式写进去，避免 agent 假设所有机器能力一致。

多机器场景我用两层结构：`tools.md` 放公共内容（进 git），`tools.local.md` 放本机特有的路径和开关（gitignore 掉），加载时 local 覆盖 common。公共文档可以放心提交，本机差异互不污染。

## 踩坑点

- **别写成操作手册**。每条控制在几行内，上下文很贵，长文档反而降低调用准确率。
- **别在 tools.md 里定义工具**。它只描述事实，注册仍在插件/MCP 配置里；两处不一致时 agent 会信文档然后翻车，改配置必须同步改文档。
- **别写密钥**。tools.md 会进上下文，等于进了日志和模型输入。
- **文档会过期**。过期条目让 agent 按文档调用必然失败；自检命令就是防漂移手段，跑一遍就能暴露。
- **Windows 机器**注意换行符和路径分隔符，条目里直接写明用哪种 shell 执行。

## 可复用建议

1. 把 tools.md 当"这台机器的 README"维护：只写事实，不写意图；
2. 每个条目必须带自检命令；
3. common + local 两层，一台机器一份 local；
4. 环境大改（升级系统、换 Python、加 MCP server）后，先人工过一遍 tools.md 再让 agent 上岗；
5. 某个条目三个月没被调用过，删掉——文档也要做减法。

## 总结

tools.md 的价值不在"写得多"，而在"写的都是这台机器的真话"。把机器差异从提示词里挪出去、把密钥挡在文档外、用自检命令对冲文档过期——这三件事做到位，同一套 agent 配置跨机器稳定运行就是水到渠成的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/dc68456c6eb20c92.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/97d92226f2902b3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d2175031cac81805.png)

