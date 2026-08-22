---
title: OpenClaw Skills 机制：把 AI 助手的能力从“全量注入”改成“按需挂载”
feedId: 34175
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

我们在 OpenClaw 上做内部研发助手时，接入了 GitLab、K8s、文档、监控、审批等能力。一开始图省事，把所有工具说明、权限声明、提示词全塞进 system prompt。功能确实都能用，但很快暴露问题：

- system prompt 膨胀到 4 万 token，每次请求都在烧钱；
- 模型开始忽略长尾工具，经常该调用的不调用；
- 不执行的工具 schema 也暴露在上下文里，权限面变大；
- 更新一个工具要改全量配置，发布像拆炸弹。

后来改用 OpenClaw 的 Skills 机制，把能力拆成按需加载的包，效果才稳定下来。这篇文章记录可复现的工程做法。

## 问题本质

全量注入的问题不是 token 多，而是“上下文里放了太多与当前任务无关的能力”。模型不是不聪明，是注意力被稀释后，长尾工具被当成噪声。

Skills 机制的核心思路：平时只放一个极简的“技能目录”，等用户需求命中某个技能时，再把该技能的 prompt、工具 schema、脚本、权限按需挂载进当前会话。

它和 MCP 不冲突。MCP 解决的是“怎么连远程工具”，Skills 更多解决“本地提示词 + 工具 + 脚本怎么打包、何时加载”。

## 做法/步骤

### 1. 定义 skill 包结构

我们约定每个 skill 是一个目录：

```
skills/
  gitlab-issue/
    skill.yaml
    prompt.md
    tools/
      gitlab_create_issue.py
    scripts/
      check_gitlab_token.py
```

### 2. 用 manifest 描述元数据

`skill.yaml` 只放声明，不放具体指令：

```yaml
name: gitlab-issue
version: 1.2.0
description: 用于在 GitLab 上创建、查询、更新 issue。当用户提到 issue、缺陷、需求单、工单时使用。
triggers:
  - issue
  - gitlab
  - 缺陷
  - 需求单
tools:
  - gitlab_create_issue
  - gitlab_list_issues
permissions:
  - read:gitlab
  - write:gitlab_issue
dependencies: []
```

`prompt.md` 里写完整指令、边界条件、错误处理。模型只在技能被加载后才看到这些内容。

### 3. 启动时建索引，不注入全文

OpenClaw 启动时扫描 `skills/` 目录，把每个 skill 的 `name + description + triggers` 建成轻量索引。系统只生成一页“技能目录摘要”，类似：

```text
可用技能：
- gitlab-issue：创建/查询 GitLab issue，用户提到 issue、缺陷、需求单时使用。
- k8s-log：查询 Pod 日志，用户提到查看日志、Pod 崩溃时使用。
```

这个摘要控制在 500 token 以内。

### 4. 会话中按需匹配

在代理层对用户输入做匹配。可以先做关键词命中，再做一次轻量语义过滤。命中后加载技能：

```python
matches = skill_index.match(user_input, top_k=2)
for skill in matches:
    if skill.name not in session.loaded_skills:
        session.context += skill.render_prompt()
        session.context += skill.bind_tools()
        session.loaded_skills.add(skill.name)
```

如果技能有依赖，先加载依赖技能，再加载主技能。

### 5. 释放与缓存

会话结束卸载全部技能。会话内已加载的技能做缓存，避免同一轮对话重复注入同一个技能的 prompt。

## 踩坑点

**description 写得太泛，误加载率高。**  
“查询信息”“处理任务”这类描述基本等于白写。好的 description 要写清楚“什么时候用”和“解决什么问题”。我们的标准模板是：`用于 {目标}。当用户提到 {触发词/场景} 时使用。`

**首轮匹配失败。**  
用户第一句经常只说“帮我提个 issue”，如果技能目录摘要里没有足够线索，匹配会失败。我们的做法：允许用户显式调用 `/skills` 查看可用技能，也能通过编号直接启用。

**依赖没声明，执行到一半才报错。**  
一个 skill 调用了另一个 skill 的工具，但 manifest 里没写 `dependencies`。结果工具绑定不上，模型开始瞎编。后来强制依赖预加载，问题消失。

**加载缓存命中旧版本。**  
一个技能更新了 prompt，但会话缓存里还是旧内容。我们把 `version` 放进缓存 key，发布新版本后旧缓存自然失效。

**权限粒度太粗。**  
一个 skill 同时有 read 和 write 权限，按需加载解决不了越权。后来在工具级别做 action 权限拆分，写操作单独确认。

## 可复用建议

- 声明与实现分离：`skill.yaml` 只写“什么时候用”，`prompt.md` 写“怎么用”。
- 技能目录摘要必须极简，一行一个技能，否则摘要本身又变成噪声。
- 给每个 skill 配 triggers，并定期回归误载率和漏载率。
- 加载动作留日志，统计命中率、token 节省量、失败 case，用数据调阈值。
- 把 skill 当版本化制品管理，变更走 review，支持灰度发布。
- 先从 2-3 个高频能力试点，跑通加载、缓存、权限链路后再批量迁移。

## 总结

OpenClaw Skills 机制不是银弹，它的收益来自两件事：元数据质量和加载策略。机制本身只是把“永远在场的工具说明”变成了“按需拉取的资源包”。如果 description 写不好、权限拆不细、缓存管不好，Skills 只会带来新的复杂度。

但做对之后，收益很直接：system prompt 从 4 万 token 降到几千 token，响应更快，工具调用更准，权限面也更小。建议把它当作 Agent 能力治理的一部分，而不是一次性接入动作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/48241ed523516d38.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/8c9abd5f2070e518.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/45781aead13310be.png)

