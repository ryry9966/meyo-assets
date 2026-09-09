---
title: OpenClaw 的 Sandbox 安全模型拆解：为什么 Agent 不会误删你的文件
feedId: 36754
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 这类本地优先的 agent 网关，默认会给模型一组很有分量的工具：shell 执行、文件读写、定时任务。能力越大，边界问题越突出——误删文件从来不是“模型不听话”，而是系统设计问题。默认的安全边界不应该依赖模型自觉。

## 问题在哪

让 LLM 直接对宿主机执行命令，风险有三类：

1. **幻觉与路径拼错**：`rm -rf` 后面跟错一个变量，就是事故。
2. **提示注入**：agent 读到的网页、邮件、issue 内容里可能藏着恶意指令。
3. **权限过宽**：进程能碰整个 home，一次失误的爆炸半径就是整个 home。

## OpenClaw 的做法：三层边界

以常见的 Docker sandbox 模式为例，边界由三层叠加构成：

1. **容器沙箱**：sandbox 开启后，agent 的 exec、写文件等工具调用被送进 Docker 容器执行。容器里的“电脑”不是你的电脑。
2. **最小挂载**：只把 workspace 目录 bind mount 进容器。容器内看到的文件系统，只有 workspace 是真的，其余是镜像里的干净环境。容器里 `rm -rf /` 删的是容器可写层，宿主机无感。
3. **网关仲裁**：所有工具调用经过 gateway，sandbox 策略在这一层统一生效；越出沙箱范围的请求要么被拒绝，要么走明确的确认流程。

System prompt 里的操作规范当然也有，但设计上不把它当防线。

## 验证步骤（可复现）

1. 配置开启 sandbox，确认本机 Docker 可用，重启 gateway。
2. `docker ps` 找到沙箱容器。
3. `docker inspect <container>` 查看 Mounts：确认只挂了 workspace，没有 `/`、`/home`、更不能有 `docker.sock`。
4. 故意让 agent 执行 `rm -rf ~/test-sandbox`，或向宿主机路径写文件。
5. 回宿主机验证：文件还在；进容器看，删除只发生在容器内部。

这套测试建议当成冒烟测试，每次大版本升级后跑一遍。

## 踩坑点

- **挂载贪大**：图省事把整个 home 挂进去，沙箱形同虚设。只挂项目目录。
- **docker.sock 进容器**：等于把宿主机控制权交出去，最经典的逃逸路径。
- **symlink 逃逸**：workspace 里若有指向宿主机敏感目录的软链，容器内删除会穿透到宿主机。定期 `find workspace -type l` 排查。
- **sandbox 关闭模式**：为了访问宿主机工具关沙箱时，确认流程和备份必须补位，别裸奔。
- **配置漂移**：改了挂载配置没重启容器，实际状态和配置文件不一致。改完记得完整 down/up。

## 可复用建议

- 每个 agent、每个项目独立 workspace，不共享大目录。
- workspace 初始化为 git 仓库，让 agent 在破坏性操作前自动 commit，相当于内置回收站。
- 审计日志保留工具调用的原始参数，出问题能精确回放。
- 把“尝试删挂载外文件”写成例行动作，验证边界持续生效。

## 总结

“Agent 不会误删文件”，不是因为模型聪明，而是因为它的默认世界里不存在不该删的东西。OpenClaw 把信任边界从提示词挪到了容器边界 + 挂载白名单 + 网关仲裁，三层缺一不可。做法本身不复杂，难的是忍住不把整个 home 挂进去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/dc2f3b62ee4a9b7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f00496f5a65bb6a9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/270257aec8f75f9f.png)

