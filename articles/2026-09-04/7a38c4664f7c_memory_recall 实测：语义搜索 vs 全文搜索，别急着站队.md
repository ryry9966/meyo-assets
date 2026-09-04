---
title: memory_recall 实测：语义搜索 vs 全文搜索，别急着站队
feedId: 36083
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 的 memory_recall 是 agent 回查历史记忆的入口：会话笔记、踩坑记录、用户偏好存在本地 SQLite/markdown 里，靠检索召回再喂回上下文。检索质量直接决定 agent 是"真记得"，还是一本正经地编。

## 问题

实现上常见两条路：全文检索（FTS5/BM25）和语义检索（embedding 向量）。教程普遍说"语义更智能"，但我们在几台长期运行的实例上实测下来，两条路各有一条明确的舒适区，单选任何一边都会在特定 query 上翻车。

## 做法

我们目前的形态是 hybrid，演进顺序如下：

1. **先上 FTS5**。SQLite 自带、零依赖。但 tokenizer 必须换成 trigram，否则中文基本不可用：

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content, tokenize='trigram'
);
```

2. **语料过百、出现"换个说法就搜不到"的 case 后，再上 embedding**。查询时把 query 一起 embed，余弦取 top-k。短查询（少于 4 个字）直接走全文，短文本的 embedding 质量很差。

3. **两路结果用 RRF 合并**，不做分数混算：`score = Σ 1/(60 + rank)`。两边分数量纲完全不同，直接加权相加是新手大坑。

4. **加两个偏置**：时间衰减（近期记忆略加权）和来源标记（长期笔记 / 会话摘要），召回时分开标注。

5. **最终召回控制在 5 条以内**，每条带时间戳进上下文。

## 踩坑点

- **中文 tokenizer 是最大的坑**。FTS5 默认 unicode61 会把连续 CJK 字符当成一个 token，不换 trigram 等于没建索引，搜索永远空结果。
- **错误码、文件路径、命令行这类精确 token，全文检索完胜**。embedding 会把 `ACCESS_DENIED` 和"权限体系设计"的讨论混在一起召回。
- **"上次那个部署方案为什么放弃了"这类意图型查询，全文检索抓瞎**，只有语义能接住。
- **语料少于 100 条时，语义检索是负资产**，向量噪声会把精确命中的结果挤出 top-k。
- **召回条数太多反而污染上下文**，agent 会顺着无关旧笔记跑偏，宁可少而准。

## 可复用建议

- 别一步到位上向量库。先 FTS5 + trigram，把"该召回却没召回"的真实 query 记下来，攒满 20 条再决定加不加语义，让数据说话。
- hybrid + RRF 几十行就能写完，是当前最稳的默认配置，不必引重型向量库。
- 语义模型换版本必须重建索引，旧向量和新 query 不在同一个空间，相似度会整体漂移。
- 每周扫一遍 recall 日志，看 top-1 命中率——比任何公开 benchmark 都真实。

## 总结

没有谁绝对更好：精确标识符靠全文，模糊意图靠语义，中文场景 trigram 是前置条件。我们的默认答案是 hybrid + RRF + 条数上限，然后用真实 query 集持续校准——而不是相信任何一方的"更智能"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/da333b8b6fd30eba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/30bd0355a8b0fe7e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c09ae19f4fee6025.png)

