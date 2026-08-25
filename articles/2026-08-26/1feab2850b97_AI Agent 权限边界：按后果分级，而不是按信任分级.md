---
title: AI Agent 权限边界：按后果分级，而不是按信任分级
feedId: 34771
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景
OpenClaw 的 Agent 很容易从“只读问答”长成“能改文件、发请求、跑命令”的执行体。接 MCP/插件后，权限分散在工具、进程、环境变量和系统账号里。一个浏览器插件能读 Cookie，一个 shell 工具能删目录，一个发送接口能把草稿推到生产。权限边界不清晰时，模型倾向“多做”，人类疲劳后倾向“全点允许”。

## 问题
核心不是“AI 能不能做”，而是动作失败后能否安全回滚、快速定位、是否愿意承担后果。可以按一句话判断：

- 只读且能回滚：自己干。
- 可逆但有副作用：可以干，必须有审计。
- 不可逆、涉及资金/发布/外部发送：必须问人类。
- 影响范围不确定：先 dry-run 或缩小范围。

## 做法/步骤

### 1. 按后果分级，不按工具名称分级
同一个 HTTP 工具，GET 和 POST 风险不同；同一个文件工具，写 /tmp 和写 ~/.ssh/authorized_keys 也不是一回事。建议分四级：

1. READ：读取、查询、搜索。
2. WRITE_LOCAL：写临时目录、草稿、可覆盖缓存。
3. EXTERNAL_REVERSIBLE：发可撤回消息、创建工单、提交草稿到 staging。
4. IRREVERSIBLE：转账、发布生产、删除用户数据、发送外部通知。

默认 1、2 级可自动执行；3 级需白名单资源；4 级必须人工确认。接 MCP 时，应在工具包装层把资源范围显式限定，而不是把整个 MCP server 无差别放进来。

### 2. 默认拒绝，授权到“资源 + 动作 + 参数范围”
很多越权不是工具选错，而是权限声明太宽。一个 shell 插件配置成 `allow_all: true`，模型就能从查日志变成改系统服务。更稳妥的配置是把权限写成最小三元组：

```yaml
permissions:
  - action: file_read
    path_prefix: ["/workspace", "/tmp/openclaw"]
    allow: true
  - action: shell_exec
    command_patterns: ["git status", "ls -la /workspace"]
    allow: true
  - action: http_post
    host_suffix: ["staging.example.com"]
    require_confirm: true
  - action: http_post
    host_suffix: ["api.example.com"]
    allow: false
```

这样不是“管住模型”，而是收窄运行环境默认能力。MCP server 尤其要注意 ambient authority：它通常继承启动进程的用户权限。如果 MCP server 能访问整个 home 目录，模型只是通过它间接获得能力。隔离目录、单独 token、只读挂载比限制提示词有效。

### 3. 人工确认只放在关键节点
如果每一步都弹确认，人类会变成审批机器人，最后闭眼允许。人工确认应集中在：不可逆动作前；参数超出预授权范围时；目标环境不是 staging/dev 时；同一动作重复执行但影响对象变化时。同会话内、同参数、已确认过的低风险动作可设短时预授权。确认必须来自外部系统，不能让模型自己生成“是否允许”。

### 4. 让 Agent 提交“确认包”，而不是问“可以吗”
模型问“我可以继续吗”没有用，因为人类不知道它到底要干什么。可以把确认动作做成结构化 require_confirm，至少包含动作、目标、影响范围、回滚方式。例如：
`POST /orders/{id}/cancel`，影响单个订单，会发送通知，不可逆，回滚需重新创建订单。
这比“我要取消订单，可以吗”更适合工程判断。

### 5. dry-run、审计和回滚
不可逆操作尽量先 dry-run。dry-run 要真实返回副作用预测，而不是让模型说“我模拟过”。发送、发布、删除类操作可接入 staging、草稿箱或软删除。每次工具调用至少记录时间、会话 ID、工具名、参数、是否经人工确认、返回状态。出现问题能按会话 ID 还原链路。

## 踩坑点
- 权限太细导致用户疲劳，用户养成随便点允许的习惯。
- 只限制工具名，不限制参数，一个 run_command 等于没有边界。
- dry-run 没有回传模型，模型没有根据结果调整下一步。
- 把确认和推理混在提示词里，模型会“说服自己”风险不高。
- MCP server 权限过大，带着整个文件系统、网络和用户环境变量。
- 没有预算和速率限制，循环调用放大已批准的权限。

## 可复用建议
- 权限矩阵外置，放在配置或策略文件里，不要散落在 system prompt。
- dev/staging/prod 三套权限配置，生产环境只开放最少工具。
- 对发送、发布、花钱、删数据四类动作要求二次确认。
- 为每个确认包和工具调用生成 request id，日志可追踪。
- 定期 review allow 和 require_confirm 规则，关掉不再需要的白名单。
- 准备熔断开关：一键暂停所有写操作，只保留只读。

## 总结
权限边界不是“AI 可不可信”的哲学问题，而是接口设计问题。工程上最简单的做法是：默认拒绝、按后果分级、确认只放在不可逆节点、把影响和回滚说清楚、所有动作可审计。这样 OpenClaw 的 Agent 能在“自己干”和“问人类”之间找到稳定边界，而不只是靠用户每次紧张地点允许。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/70624295faab3875.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/51cc20cb54b5f514.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/4634c0ece9b95edf.png)

