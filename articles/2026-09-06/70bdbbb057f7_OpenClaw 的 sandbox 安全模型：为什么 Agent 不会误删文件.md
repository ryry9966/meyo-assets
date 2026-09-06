---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36348
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

给 Agent 接上 shell 和文件工具之后，第一件让人睡不着觉的事就是误删。OpenClaw 的设计前提是：模型能力越强，越不能靠"提示词求它小心一点"来兜底。所以 sandbox 不是可选插件，而是执行链路的默认组成部分。

## 问题：误删到底怎么发生的

复盘真实事故，误删几乎都不是模型"想删"，而是三类因素叠加：

1. **路径理解偏差**：相对路径、工作目录漂移、`~` 展开，agent 心里的目录和真实目录不一致；
2. **工具权限过大**：一个能跑任意 Bash 的会话，等价于把整台机器交给概率；
3. **缺少不可逆闸门**：rm、git clean、覆盖写没有第二道确认。

## 做法：三层模型

OpenClaw 的 sandbox 分三层，可以只开第一层，但生产建议全开。

**第一层：文件系统 jail。** 默认只把 workspace 以 bind mount 挂进沙箱，其余路径对进程不可见——不是"不让写"，是根本看不见。

```yaml
sandbox:
  mounts:
    - host: ./workspace
      guest: /workspace
      mode: rw
    - host: ~/.cache/pip
      guest: /root/.cache/pip
      mode: ro
```

**第二层：能力声明。** 每个 MCP 工具、插件注册时声明 capability（fs.write、fs.delete、net.exec 等），策略层按会话放行。工具没声明 fs.delete，就不会出现在 agent 的工具列表里——不是拦截报错，是压根不知道有这功能。

**第三层：破坏性命令闸门。** 命中规则（rm -rf、`>` 覆盖、git clean）的调用先走 dry-run：OpenClaw 先算出"会删什么"的 diff，超范围直接拒绝，范围内且开启 confirm 才放行。

验证方法：部署后故意构造一条"把 ~/.ssh 清理一下"的测试指令，预期结果是 agent 在沙箱内找不到该路径并如实报告，而不是静默失败。

## 踩坑点

- 挂了整个 `$HOME` 图省事，等于 jail 白设，坚持挂最小集合；
- 把 Docker socket 挂进沙箱：agent 通过 `docker run -v` 反手把宿主机全挂进来，jail 形同虚设；
- Bash 兜底绕过：文件工具被拦，agent 改用 python 写文件。沙箱必须作用在进程层（文件系统视图），而不是工具层的关键字过滤；
- symlink 逃逸：workspace 里早先留下的软链接指向外部，bind mount 不主动拦，启用 resolve+deny 策略；
- 第三方 MCP 工具常默认申请过宽 capability，每装一个新插件都要重新过一遍策略。

## 可复用建议

1. 只读是默认，写是例外；
2. 删除类操作永远过 diff 闸门，代价是几百毫秒，收益是不可逆操作全程有审计记录；
3. 每次升级插件跑一遍"越狱测试集"，当作回归测试的一部分；
4. 审计日志里保留 dry-run 的 diff，事后才能回答"它当时到底想删什么"。

## 总结

"Agent 不会误删文件"不是模型听话，而是架构上让误删不可表达：看不见的路径删不了，没声明的工具调不到，声明过的删除必须先出示清单。信任来自边界，不来自祈祷。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/65edf0b528353918.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/b23cc4aff5946da5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/5458045efc7641cc.png)

