---
title: AI 助手的 memory_recall：语义搜索 vs 全文搜索，实测怎么选
feedId: 35679
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 的记忆召回依赖 workspace 里的 markdown 记忆文件，加一层 SQLite 索引。索引里有两条检索路径：**语义**（embedding 向量）和**全文**（FTS5 关键词）。agent 每次调 memory 工具搜索时，走哪条路径、权重怎么配，直接决定它“想不想得起来”。最近我在自己的 agent 上做了一轮对比，把结论沉淀成这篇。

## 问题

起因是 agent 明明记过我的部署偏好，追问时却说找不到相关记忆。排查后发现语义检索根本没生效，静默回退到了纯全文搜索——而我的提问全是中文口语，关键词对不上，于是“失忆”。这引出两个问题：两种检索各自擅长什么？怎么配置和验证？

## 做法

1. **先看状态**：`openclaw memory status`。重点看 embedding 是否加载、索引是否新鲜。很多“召回不准”其实是语义路径没跑起来。
2. **配置 embedding provider**：可以走 OpenAI，也可以指向本地 Ollama 的 OpenAI 兼容端点：

```jsonc
{
  "agents": { "defaults": { "memorySearch": {
    "provider": "openai",
    "remote": {
      "url": "http://127.0.0.1:11434/v1",
      "apiKey": "ollama",
      "model": "nomic-embed-text"
    }
  }}}
}
```

（字段名以你当前版本文档为准。）

3. **改完必须重建索引**：`openclaw memory index --force`。
4. **用同一组查询做对比**：`openclaw memory search "..."`，我测了两类：
   - 精确串类：错误码、命令名、文件名——全文检索稳赢，语义反而可能排不上；
   - 换说法类：中文口语、同义改写——语义检索稳赢，全文常常空结果。
5. **打开 hybrid 混合检索**，让两路分数加权融合，再按需调 maxResults / minScore。

## 踩坑点

- 没配 embedding 时**不会报错**，只会静默退化为全文。判断依据看 `memory status`，别靠感觉。
- 换 embedding 模型后不重建索引，向量维度和语义空间全错位，召回质量悄悄下滑，很难第一时间察觉。**改模型 = 重建索引**，固化成肌肉记忆。
- FTS5 默认分词对中文不友好，长句基本靠命。如果 memory 主要是中文，语义路径更要配好。
- minScore 设太高，混合模式下会先把全文命中的结果过滤掉，表现为“明明写了却搜不到”。
- 远程 embedding 有延迟和费用；本地模型首次建索引慢，好在有缓存，别动不动全量重建。
- 检索调优有上限：memory 文件本身写得乱，什么检索都救不回来。

## 可复用建议

- 默认 hybrid，别二选一。语义管“意图”，全文管“精确 token”，互补而非替代。
- 写 memory 时一条笔记同时带“关键词 + 自然语言描述”，两路检索都好命中。
- 攒一个 10 条左右的真实提问回归集（从自己的会话日志里挑），每次改配置就跑一遍，避免“感觉变好了”。
- 升级或更换 provider/model 后，`index --force` + `memory status` 两连。
- 定期归档过期的 daily notes，控制噪声。

## 总结

语义搜索解决“换个说法还记得”，全文搜索解决“精确串必命中”，任何单一方案都会在你最需要时掉链子。工程上更靠谱的姿势是：hybrid 打底、embedding 状态常检查、模型变更必重建、用真实 query 做回归。先确认检索真的在跑，再谈调优。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/84a5bad913bcb13a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/32a757060379f674.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/21ac4080a3ef780c.png)

