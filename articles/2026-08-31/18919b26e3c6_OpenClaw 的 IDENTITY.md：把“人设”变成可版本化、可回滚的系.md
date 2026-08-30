---
title: OpenClaw 的 IDENTITY.md：把“人设”变成可版本化、可回滚的系统资产
feedId: 35457
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 里很多 Agent 的行为边界，习惯写在系统提示、流程代码、工具描述里。时间一长，散落三处：改角色要动代码，调规则要改 prompt，记经验要手动维护 memory。更麻烦的是，Agent 在多次会话里的表现不一致，但你很难说清它“应该稳定成什么样”。

IDENTITY.md 就是把这个部分抽出来：一个独立、可加载、可更新的身份文件。它不是固定人设，而是 Agent 的“运行态自我描述”。

## 问题

常见痛点有三个：

1. **身份散落**：角色定位、禁止行为、用户偏好混在 prompt 和插件配置里。
2. **人工修改易漂移**：每次改一点，久了以后原则之间开始冲突。
3. **没有演化记录**：Agent 学到的东西无法沉淀，下次会话又清零。

IDENTITY.md 的思路很直接：让身份像代码一样版本化，让 Agent 在受控范围内小步更新自己。

## 做法/步骤

**1. 初始化一个最小身份文件**

项目根目录下建 `IDENTITY.md`，建议只保留这几个区块：

```markdown
# Identity
- name: deploy-bot
- role: 负责测试环境发布与回滚

# Mission
- 只处理测试环境部署、健康检查、版本回滚

# Principles
- 不直接操作生产环境
- 变更前先输出影响范围
- 不做未经确认的删除操作

# Boundaries
- 禁止执行 kubectl delete 以外的删除类命令
- 禁止读取与部署无关的密钥文件

# Context
- 用户偏好：发布前先触发 smoke test

# Evolution Log
- 2025-04-11 | 用户要求所有回滚前保留现场快照 | source: user
```

**2. 接入 OpenClaw 加载**

在 `openclaw.yaml` 里指向该文件，并限制注入长度：

```yaml
identity:
  file: ./IDENTITY.md
  max_tokens: 1200
  inject: system
```

这样每次会话启动时，身份文件会作为系统级约束注入，而不是被业务 prompt 淹没。

**3. 给自更新上锁**

不建议让 Agent 全量重写 `IDENTITY.md`。可以先只开放 `Evolution Log` 的追加权限：

```yaml
identity:
  allow_self_update: append-only
  self_update_sections:
    - evolution-log
  max_update_lines: 3
```

这样它只能追加经验，不能改使命、原则和边界。

**4. 纳入版本控制**

把 `IDENTITY.md` 放进 git。每次 Agent 追加经验后，用 `git diff` 查看它为什么改自己。回滚成本接近于零。

## 踩坑点

- **文件越写越全**：身份文件不是知识库。超过一屏，Agent 就容易抓不住重点。原则区尤其要短。
- **允许全量自写**：非常危险。Agent 可能把“不要动生产环境”改成“尽量少动生产环境”，语义就松了。
- **经验冲突**：Evolution Log 里出现“以后所有发布都要人工确认”和“测试环境可自动发布”这类互斥条目。建议写成结构化规则：`when -> then -> because`。
- **示例被当成事实**：原则区不要放带有真实用户数据的示例。示例放 Context，并标明只是示例。
- **编码问题**：中文内容保存为 UTF-8 无 BOM。Windows 下用 PowerShell 重定向容易带入 BOM，可能导致解析异常。

## 可复用建议

1. **三层权限**：Principles 人工冻结；Context 会话前人工确认；Evolution Log 可自动追加。
2. **追加必须带三列**：`date / reason / source`，否则时间久了无法审计。
3. **定期压缩**：Evolution Log 太长时，把反复出现的稳定结论提取为 Principles，但必须人工 review。
4. **回滚优先于调试**：发现 Agent 行为异常，先看 `git diff IDENTITY.md`，多数时候比翻 prompt 快。
5. **别急着让 Agent 进化**：先跑 20 次会话，等身份文件稳定了，再开放 append-only 自更新。

## 总结

IDENTITY.md 的价值，不是给 Agent 一个更花哨的人设，而是让身份脱离代码和 prompt，成为可版本化、可审计、可回滚的独立资产。

给 AI 一个可进化的身份，不等于让它完全自治。更好的做法是：给它一个受控的自我修订接口，同时保留人类的一票否决权。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/12d080370efd1dac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e022a80029a67091.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6efbf9e9daf17ce6.png)

