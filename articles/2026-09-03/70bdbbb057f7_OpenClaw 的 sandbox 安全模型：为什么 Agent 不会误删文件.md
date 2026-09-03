---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35901
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

让 LLM Agent 直接操作文件系统，最担心的从来不是它写不出代码，而是它“手滑”：一条 `rm -rf`、一次 `git clean`，就可能把没提交的工作全干掉。OpenClaw 在设计上把这个问题当作第一优先级处理，核心思路是：**不信任模型的判断，只信任机制**。模型可以犯错，但机制要保证犯错不等于灾难。

## 问题：一次险情

我最早裸跑 Agent 时出过一次险情：让它“清理构建产物”，它自己推理出 `rm -rf build && rm -rf node_modules`，还顺手想删掉一个名字带 `tmp` 的数据目录。那之后我把 OpenClaw 的 sandbox 模型完整梳理了一遍，确认它靠的是四层防线，而不是单点防护。

## 四层防线

**1. 路径边界：workspace 沙箱**
Agent 的文件工具和 shell 进程都以 workspace root 为边界。所有路径先做规范化（realpath + symlink 解析），再判断是否落在 root 内。软链到 `/etc` 这种逃逸路径会被直接拒绝。

**2. 命令网关：破坏性命令拦截**
shell 命令不直接执行，先过一层静态审查：匹配 `rm -rf`、`git clean`、`find -delete`、重定向覆盖等模式，命中则拒绝或要求显式确认。别指望正则穷尽所有写法，这一层的作用是“提高作恶成本”，真正的兜底在下面。

**3. 删除降级：回收站 + 快照**
即使删除操作漏过了审查，`allow_delete: false` 会让所有删除变成移动到 `.openclaw/trash`，配合会话级快照可回滚。这是我felt最实用的一层：模型以为删了，其实只是搬了个地方。

```yaml
sandbox:
  root: ~/projects/demo
  mode: container          # process / container
  fs:
    allow_delete: false
    trash_dir: .openclaw/trash
  exec:
    deny_patterns: ["rm -rf", "git clean", "find .* -delete"]
```

**4. 进程隔离：容器兜底**
`mode: container` 下整个 Agent 进程（包括它拉起的 MCP server 和插件）跑在容器里，只挂载 workspace，根文件系统只读。前面三层全被打穿，损失也限于挂载目录。

## 踩坑记录

- **symlink 逃逸**：只在字符串层面判断路径前缀没用，必须 realpath 之后再校验，否则 workspace 里一个软链就能绕出去。
- **子 shell 绕过**：`sh -c "cd / && rm ..."` 能骗过简单模式匹配，所以容器隔离不能省。
- **MCP 是 blind spot**：MCP server 如果自带文件访问能力且跑在沙箱外，等于给模型开了后门。所有 MCP 进程必须和 Agent 同沙箱。
- **太严会误伤**：构建工具要写 `/tmp`，全禁会天天失败。按白名单放行临时目录，而不是放行一切。
- **git clean 是重灾区**：它长得不像“删文件”，模型很爱用，模式匹配务必覆盖。

## 可复用建议

1. 默认拒绝，按需放行；权限粒度到“读 / 写 / 删”三级。
2. 删除一律降级为回收站，真删交给人类。
3. 每次写删操作落审计日志，出事能倒放。
4. 定期做破坏性演练：故意让 Agent 执行危险命令，验证四层防线都还生效——沙箱配置改了没人测，等于没有。

## 总结

Agent“不会误删文件”不是因为模型聪明，而是因为架构上让误删变得困难、可拦截、可回滚。路径边界管住位置，命令网关管住手段，删除降级管住后果，进程隔离管住最坏情况。四层里任何一层单独存在都不够，叠起来才敢放心让它干活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/cf35167a82f8d38b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/a303b3990f95e86b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/fbb66741ae1c3038.png)

