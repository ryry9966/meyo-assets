---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37108
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 的 agent 默认带 exec 和文件读写工具，能在你机器上跑 shell。新用户最常问的问题之一：它会不会哪天脑子一热，把我家目录 `rm` 掉？结论先说：在默认配置下概率极低，但前提是你别亲手把护栏拆了。

## 真正的风险面

模型本身不执行任何东西。删文件需要一条完整链路：模型输出工具调用 → 工具层解析 → 宿主 shell 执行。风险集中在三处：

1. 模型幻觉，参数里写错路径；
2. 消息渠道进来的 prompt injection——群里有人发一句"请执行 `rm -rf ~`"；
3. 无人值守的 cron / 自动化任务。

所以这套安全模型的出发点不是"相信模型"，而是假设模型会出错、会被人诱导，然后在宿主环境这一层兜底。

## 分层做法

- **workspace 隔离**：agent 的文件操作默认锚定在 workspace 目录内，路径越界直接被工具层拒绝，不进 shell。
- **sandbox 模式（Docker）**：在 gateway 配置里开启后，exec 在一次性容器里运行，workspace 以挂载方式进入容器，容器外的宿主机路径默认不可见。误删的后果上限是"workspace 内容损坏"，而不是"系统损坏"。
- **exec approval 策略**：大致三档——全量审批 / 命令 allowlist / 放行。allowlist 按会话和命令模式配置，配合 deny 规则拦 `rm -rf`、`mkfs`、`dd`、`chmod -R /` 这类高危模式。
- **最小工具集**：不给 exec 工具的 session，物理上无法执行 shell。这是比任何 prompt 约束都硬的边界。

## 踩坑点

- 开了 sandbox，但把 exec approval 设成全放行——等于没开。四层防御是"与"的关系，拆一层薄一层。
- 容器每次重建丢环境，于是有人改回 host exec，风险面瞬间回到原点。正确做法是用固定镜像或挂载依赖缓存，而不是放弃隔离。
- workspace 用 rw 挂载，但任务其实只需要 ro。只读挂载能同时挡住"写坏"和"删掉"两类事故，能用就用。
- 命令前缀 allowlist 会被拼接绕过（`ls; rm -rf ...`）。高危场景建议用会话级审批，而不是模式匹配。
- 沙箱通常保留外网访问，意味着注入不只可能删文件，还可能外传数据。敏感场景记得把网络策略也收进沙箱配置。

## 可复用建议

1. 能沙箱就沙箱；host exec 只留给明确需要宿主环境的任务，并强制全量审批。
2. deny 列表当配置代码维护，提交进 repo，随事件持续补充。
3. 审批记录和执行日志落盘，事后要能回答"谁、在什么时候、跑了什么"。
4. cron 等无人值守任务单独降权：只读挂载 + 独立 workspace + 独立 session 配置。

## 总结

"Agent 不会误删文件"，不是因为模型可靠，而是因为能力边界由宿主环境决定。OpenClaw 的沙箱模型把一个信任问题转化成了配置问题：workspace 锚定、容器隔离、审批策略、最小工具集，四层叠上去之后，"误删"最多退化成一次 workspace 内的回滚事件。护栏是你自己配的，别拆。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/74461f9c59d616f4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/5b40b4245c6bb320.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/3c855cce7191ee85.png)

