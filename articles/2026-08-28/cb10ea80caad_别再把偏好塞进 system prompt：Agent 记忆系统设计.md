---
title: 别再把偏好塞进 system prompt：Agent 记忆系统设计
feedId: 34964
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

常用 OpenClaw 做自动化或插件编排的人，大概率遇到过这种循环：今天告诉 Agent “输出用中文、不要 emoji、接口报错先查日志再重试”，明天开新会话它又回到默认风格。不是模型笨，而是这些信息没有被当成可管理的状态。

MCP 让 Agent 能调用外部工具，但 memory 经常被简化为“往 system prompt 里加一段偏好文本”。会话一多，上下文被低价值描述占满，真正该注入的偏好反而被截断。所以核心问题不是“要不要记忆”，而是记忆该以什么结构进入系统。

## 问题

Agent 记忆至少面临四类问题：

1. 偏好不是单一值：有稳定偏好，比如语言风格；有任务相关偏好，比如部署前先 dry-run；也有临时上下文，比如当前项目使用 pnpm。
2. 不能只靠向量检索：偏好通常是短句，语义相似但含义可能冲突，例如“不要自动重试”和“网络错误可以重试一次”。
3. 更新比写入更难：用户改口或场景变化后，旧偏好会继续干扰。
4. 注入位置决定效果：记忆放在 system prompt、工具说明还是检索片段，差别很大。

如果不区分这些，记忆系统会很快变成噪声源。

## 做法：三层记忆 + 规则优先

我把记忆分为三层，配合一个轻量 MCP memory server 使用。

**第一层：Profile / 稳定偏好**

只存跨会话复用的低熵信息，例如语言、回答风格、常用环境路径、明确禁令。字段建议：

```json
{
  "key": "prefer_language",
  "value": "zh-CN",
  "scope": "user",
  "source": "explicit",
  "confidence": 0.95,
  "updated_at": "2026-01-01T10:00:00Z"
}
```

这类条目控制在 20-50 条以内，直接拼在 system prompt 顶部，别超过 800 tokens。

**第二层：Session / 工作记忆**

只存当前会话或当前自动化任务的临时状态，例如“本次 target_host 是 10.0.0.12”“当前分支是 feat/memory”。用键值对即可，会话结束清理，不进入长期存储。

**第三层：Episodic / 经验记忆**

存“用户曾经纠正过什么”，不是存原始对话，而是存结论。例如：

```json
{
  "rule": "当用户说“先别执行”时，只生成计划，不调用任何写操作工具",
  "trigger": ["approval_required", "write_tool"],
  "confidence": 0.8,
  "source": "correction"
}
```

这一类适合用规则引擎先匹配 trigger，再决定是否注入，而不是每次全量检索。

如果使用 MCP，可以暴露三个工具：`remember`、`recall`、`forget`。OpenClaw 插件在每次任务开始时调 `recall(scope=profile, limit=15)`，任务中命中的经验规则按需注入。

## 踩坑点

**把大段偏好文本当 memory**

最常见做法是让 Agent 自己总结偏好，再原样写回。问题是文本越长，越难更新，冲突时无法定位。

**不记录来源和时效**

“不要用 Docker”可能是针对某个内网环境说的，结果几个月后换项目还在生效。每条记忆至少带 `source` 和 `updated_at`，对超过 30 天未命中的条目降权或提醒确认。

**向量检索不是默认选项**

偏好通常是短结构化信息，确定性规则 + 关键字匹配已经能解决大部分场景。上向量库后，相似的错误偏好会被同时召回，增加仲裁成本。

**没有“忘记”通道**

用户说“以后不要这样了”应触发更新或删除，而不是新增一条。最好在对话里识别纠正语气，生成 `update` 操作，并显示给用户确认。

**把记忆当作全局共享状态**

多人共享同一个 MCP memory server 时，个人偏好可能污染团队自动化。至少用 `scope: user/project/team` 隔离，涉及密钥、账号、内网地址等不写入通用 memory。

## 可复用建议

1. 偏好用结构化条目，而不是长文描述；每条不超过 150 tokens。
2. 注入有预算：profile 注入不超过 20 条，经验规则一次不超过 5 条。
3. 每个记忆字段固定：`key/value/scope/source/confidence/updated_at`。
4. 提供显式纠正命令，如 `/forget prefer_emoji`，让清除和写入一样简单。
5. 先做规则命中，再做语义检索；规则能覆盖 80% 的场景。
6. 在日志里记录“本次注入了哪些记忆、用户是否修正”，持续观察命中率。

## 总结

让 AI 助手真正记住偏好，不是把更多内容塞进上下文，而是把记忆做成可写入、可检索、可失效、可隔离的状态系统。对 OpenClaw 这类 Agent 场景，三层记忆加一个轻量 MCP memory server，配合规则优先的注入策略，足够解决大部分实际问题。关键不是记忆容量，而是记忆的更新成本和噪声控制。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/90a4482c3afca275.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1009d2d2cc2959a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1f807974e087a2d0.png)

