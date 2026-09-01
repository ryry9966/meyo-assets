---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35684
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 的 agent 默认带 shell、文件读写、浏览器这些工具，等于在宿主机上放了一个会自己敲命令的执行器。很多人的第一反应和我一样：这不就是给桌面放了个能自主决定要不要 `rm -rf` 的东西吗？跑了几个月之后我的结论是：误删这件事，靠的从来不是模型自觉，而是 sandbox 的结构性约束。

## 问题

LLM 的输出不可预测，所以"它应该不会乱来"不能当安全边界。真正要回答的是两个问题：当模型确实生成了破坏性命令时，系统里有哪几层会拦它？万一全没拦住，损失能不能被限制在很小的范围内并恢复？

## 做法：三层防线

**第一层，边界。** 用 Docker sandbox 跑 agent：容器内只挂载 workspace 这一个可写目录，宿主的 HOME 和其他项目目录默认根本不在容器的文件系统里；需要读的配置用只读挂载；容器内跑非 root 用户。此时模型执行 `rm -rf /`，删掉的只是容器层和 workspace，物理上够不到宿主。

**第二层，策略。** tool policy 对 exec 做 allowlist 加审批：常规命令直接放行，`rm`、`sudo`、`dd`、`mkfs` 这类命中危险规则的转人工确认。注意审批要按语义做——拦 `bash -c` / `sh -c` 这类包装器，而不是只匹配 `rm` 前缀，否则一句 `bash -c "rm -rf ..."` 就绕过去了。

**第三层，兜底。** workspace 本身是 git 仓库，每轮任务结束自动 commit。前两层都失效时，损失被压在一次提交以内，`git reset` 即可恢复。

验证方式是逃逸演练：故意让 agent 删挂载点之外的路径、读 `/etc/passwd`、写 workspace 外的文件，逐一确认被拒。我第一次演练就抓到了 symlink 穿透。

## 踩坑点

- 图省事把整个 HOME 挂进容器，沙箱等于没做。
- 容器挂了 docker.sock 或开了 privileged，等于交出宿主 root。
- workspace 里的软链接指向宿主目录，写操作跟随链接穿出边界；挂载前清理，或对路径做 realpath 校验。
- exec 审批只做前缀匹配，被 shell 包装器绕过。
- 备份和 workspace 在同一块盘上，一次 `rm -rf` 一起带走。
- 多 agent 共用 workspace，A 做清理时把 B 的产出删了；按 agent 分目录。

## 可复用建议

- 默认拒绝、按需放行；每开放一个能力先问一句"它删了会怎样"。
- 能只读挂载就不要给写权限。
- 危险命令审批看语义，不看前缀。
- workspace 常态化提交，让回滚成本远低于一次误删的心理成本。
- 每次升级 OpenClaw 或改沙箱配置后重跑一遍逃逸演练——沙箱配置是会腐化的。

## 总结

Agent 不误删文件，不是因为它"懂事"，而是破坏性操作物理上够不到、策略上过不去、结果上可回滚。三层防线单独看都不算强，叠起来才构成边界。把"模型会不会乱来"这个问题换成"它乱来时会发生什么"，安全设计才算真正落地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3b8d7d7ef4bdada9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4ea2c5d391703a24.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/9d0001baef74059c.png)

