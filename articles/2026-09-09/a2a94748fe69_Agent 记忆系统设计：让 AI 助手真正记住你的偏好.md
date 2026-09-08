---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 36668
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

用 Agent 做日常自动化的人都会遇到同一个尴尬：每次新会话，助手都"失忆"。你得重新告诉它——代码用 Tabs 缩进、回复用中文、时区 UTC+8、部署先走 staging。会话级上下文解决不了这个问题，因为 LLM 本身无状态，context window 再大也不该拿来当数据库用。

## 问题

常见的三种错误做法：

1. **全塞进 system prompt**：偏好一多，prompt 膨胀到几千 token，模型开始漏读中段内容，成本也上去了。
2. **聊天记录全量 RAG**：检索回来的大多是一次性对话细节，噪音会污染真正的偏好。
3. **只写不管**：记忆条目只增不减，三个月后系统里躺着两千条互相矛盾的"事实"。

记忆系统本质是数据工程问题：schema、生命周期、冲突消解，比选什么模型重要得多。

## 做法

我的方案分四步，在 OpenClaw 里通过 MCP tools 落地：

**1. 分层**
- 长期偏好：稳定、低频变更（语言、风格、工具链）
- 项目记忆：跟仓库绑定（构建命令、目录约定）
- 会话记忆：本轮任务上下文，会话结束归档或丢弃

**2. 结构化 schema**

不存原文，存结构化记录：

```json
{
  "key": "code.style.indent",
  "value": "tabs",
  "type": "preference",
  "confidence": 0.9,
  "source": "explicit",
  "updated_at": 1730000000
}
```

存储用 SQLite 就够了。先别上向量库——偏好类记忆的 key 可以归一化，精确匹配比语义检索召回更准。

**3. 写入路径：显式 + 隐式双通道**
- 显式：用户说"记住xxx"，Agent 调 `memory_write`，source 标记为 explicit
- 隐式：后台任务定期扫描对话抽取候选。关键规则：候选以低 confidence 入库，重复出现 2~3 次才升级为正式偏好

**4. 读取路径：预算内注入**

会话启动时检索 top-k（我限制在 20 条 / 500 token 以内），优先级：key 精确匹配 > 标签匹配 > embedding 相似度。注入前先合并去重。

## 踩坑点

- **一次性指令被存成永久偏好**。"这次用 Python 写"被隐式通道存成"永远用 Python"。解法就是 confidence 升级机制，explicit 永远压过 implicit。
- **冲突没策略，旧值盖新值**。按 updated_at 取最新 + source 加权；写入前先查同 key 旧记录做 merge，不要 blind insert。
- **全量注入**。超过 50 条模型明显开始漏读。宁可少注入，让 Agent 在需要时主动调 `memory_search`。
- **记忆对用户不可见**。存了什么不知道，错了没法改，信任直接崩塌。务必提供 `memory_list` / `memory_delete`，让用户能查看和清理。

## 可复用建议

1. 从 SQLite + 单表 + 50 条注入上限起步，跑两周再加检索复杂度
2. 记忆必须可枚举、可编辑、可删除——这是信任问题，不只是技术问题
3. 加一个度量：统计"用户重复声明同一偏好的次数"，这个数字下降说明系统在生效
4. TTL 别省。项目记忆随仓库生命周期过期，长期偏好也要定期让用户 review

## 总结

让 Agent 记住偏好，难点不在"存"，而在写入过滤、冲突消解、注入预算，以及把控制权留给用户。先用最土的结构化方案跑通闭环，再谈 embedding 和个性化检索。工程上靠谱的记忆系统，往往朴素得让你失望——但它真的能用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/ac0dd95cbad8352b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/6199b90905aba2d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/fd151c0619cb1487.png)

