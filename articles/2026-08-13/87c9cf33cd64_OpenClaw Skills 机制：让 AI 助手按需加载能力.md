---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 32853
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在 OpenClaw 这类 Agent 框架里，能力通常来自几个方向：系统提示词、内置工具、MCP 工具、脚本插件。很多实践会把所有能力一次性注册进上下文和工具列表，结果就是：

- system prompt 越来越长，token 消耗稳定上升；
- 工具定义互相重叠，模型容易选错工具；
- 调试时很难判断某个能力为什么被触发；
- 权限面扩大，所有能力常驻意味着所有能力都可能被误调用。

Skills 机制解决的不是“能不能做”，而是“什么时候把能力加载进来”。它把能力拆成独立单元，平时只暴露轻量索引，模型判断需要时再加载完整技能定义。本质上是上下文工程 + 工具生命周期管理。

## 问题

全量常驻带来的典型问题包括：

1. **上下文膨胀**：几十个 MCP 工具和插件说明全部塞进 prompt，真正有效信息被稀释。
2. **工具冲突**：多个工具功能相近，模型面对模糊指令容易选错。
3. **切换困难**：不同项目、不同环境需要不同能力，全量加载无法按场景裁剪。
4. **安全边界模糊**：不用的能力也保持可调用，增加误操作和注入风险。

按需加载的核心目标，是把“能力索引”和“能力实现”分离：索引足够轻，可以常驻；实现足够完整，但只在命中时进入上下文。

## 做法/步骤

### 1. 组织技能目录

一个 Skill 通常就是一个目录，里面包含技能说明、脚本和引用文档：

```
skills/
  gitlab-mr/
    SKILL.md
    scripts/create_mr.sh
    references/gitlab_api.md
```

`SKILL.md` 是入口，脚本和引用文档不直接进上下文，由技能步骤按需调用。

### 2. 写 SKILL.md 元数据

`SKILL.md` 头部用 frontmatter 声明触发条件、能力和工具依赖：

```markdown
---
name: gitlab-mr
description: Create and review GitLab merge requests.
when_to_use: User wants to open, create, or review a merge request on GitLab.
tools: [bash, gitlab]
---
```

正文只写最小执行步骤，例如：

```markdown
## Steps
1. Confirm source branch and target branch.
2. Run scripts/create_mr.sh with branches as arguments.
3. Post the MR link back to the user.
```

### 3. 启动时生成索引

OpenClaw 扫描 `skills/` 目录，读取每个 `SKILL.md` 的 frontmatter，生成类似这样的轻量索引：

```text
- gitlab-mr: Create/review GitLab merge requests.
- k8s-log: Fetch and analyze Kubernetes pod logs.
- jira-ticket: Create and update Jira tickets.
```

会话中只注入这份索引，不注入技能正文。

### 4. 命中后加载

当用户说“帮我提个 MR”，模型根据索引匹配到 `gitlab-mr`，OpenClaw 再把对应的 `SKILL.md` 内容加载进当前 session。技能执行完成后，可以卸载或标记过期，避免长期占用上下文。

### 5. 设置生命周期

建议给加载的技能加 TTL 或任务级作用域。例如任务结束、用户切换话题、超过 N 轮未使用时，自动卸载技能正文，只保留索引。

## 踩坑点

1. **描述写得太泛**  
   `when_to_use` 如果写成“when user needs help”，基本等于没有索引。要写具体动作、对象、场景，比如“when user wants to create a merge request on GitLab”。

2. **技能正文过长**  
   把完整 API 文档塞进 `SKILL.md`，加载后上下文立刻膨胀。长文档放 `references/`，正文只留执行步骤。

3. **脚本环境变量缺失**  
   技能脚本依赖的环境变量在 Agent 进程里不一定存在。建议注入 `SKILL_DIR` 和明确的工作目录，脚本内部使用绝对路径，不要依赖启动时的相对路径。

4. **多个技能同时命中**  
   用户说“创建任务”可能同时匹配 Jira 和本地 TODO。需要给每个技能写清排他触发条件，必要时在描述里加“not for...”边界。

5. **权限过大**  
   Skills 不等于免审批执行。对写操作、删除操作、外网请求要设置确认策略，避免 prompt 注入或误触发。

6. **状态污染**  
   已加载技能如果一直留在 session 里，后续对话可能被旧指令干扰。卸载策略要和任务边界绑定。

## 可复用建议

- **description 用三段式**：触发条件 + 动作 + 结果，例如“when user asks to create MR / create MR via script / return MR link”。
- **单一职责**：一个 Skill 只做一件事。复杂流程拆成多个技能，或者一个技能内部用明确 steps 串联。
- **正文保持克制**：`SKILL.md` 控制在 50–80 行左右，长内容放 `references/`。
- **建立回归问题集**：准备一组用户指令，测试不同表述能否命中正确技能，记录误触发和漏触发。
- **观测加载数据**：记录每个技能的加载频次、平均上下文增量、失败率。高频技能考虑转成固定工具，低频技能保持按需加载。

## 总结

OpenClaw Skills 机制的核心不是增加一个插件目录，而是给能力加上“索引—加载—执行—卸载”的完整生命周期。它的收益体现在三处：上下文更干净、工具选择更准确、权限边界更可控。

落地时要克制：先小范围试点，控制技能数量，严格写触发条件，明确卸载规则。按需加载只有在索引准确、实现精简、边界清晰时，才会真正降低 Agent 的长期维护成本。

---

