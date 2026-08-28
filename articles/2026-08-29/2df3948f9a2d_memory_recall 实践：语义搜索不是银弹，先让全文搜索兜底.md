---
title: memory_recall 实践：语义搜索不是银弹，先让全文搜索兜底
feedId: 35150
source: 综合讨论
publishedAt: 2026-08-29
---

# memory_recall 实践：语义搜索不是银弹，先让全文搜索兜底

## 背景
在 OpenClaw Agent 里做长期记忆时，memory_recall 通常被做成一个 MCP 工具：Agent 把用户偏好、项目约定、历史决策、踩坑记录写入存储，下次任务前先调用 recall 取回相关记忆。但召回质量直接决定记忆系统是“有用”还是“干扰”。

一开始很容易默认上向量检索：把每条记忆做 embedding，query 做相似度匹配。实际跑一段时间会发现，纯语义搜索在工程场景里经常失准；而数据库自带的全文搜索（FTS）反而更稳，成本低、可解释、好排查。

## 问题：两种搜索的真实边界
全文搜索适合：
- 错误码、命令、路径、函数名、专有名词；
- 版本号、配置项、日志关键词；
- 用户明确说过的原话，如“api.foo.com/login 超时 30s”。

语义搜索适合：
- 同义改写，比如“登录接口慢” vs “认证超时”；
- 意图模糊，比如“之前怎么解决数据库连接问题”；
- 跨语言或不固定措辞的自然语言查询。

二者不是替代关系。纯语义搜索容易把“MySQL 连接超时”召回成“Redis 连接超时”，因为它们向量距离很近；纯全文搜索又召回不了换了个说法的相同经验。

## 做法/步骤
我在 OpenClaw 里把 memory_recall 做成混合模式，默认 hybrid，可手动切 fts / semantic。核心存储用 SQLite：

1. 建主表
```sql
CREATE TABLE memories (
  id INTEGER PRIMARY KEY,
  content TEXT NOT NULL,
  scope TEXT NOT NULL,          -- 项目/会话/工具域
  source TEXT,
  updated_at TEXT NOT NULL
);
```

2. 建 FTS5 表
```sql
CREATE VIRTUAL TABLE memories_fts USING fts5(
  content,
  tokenize='unicode61'
);
```
中文场景如果 SQLite 编译支持 trigram，可以用 `tokenize='trigram'`，否则在写入前用 jieba 分词，把空格分隔文本写入 FTS 表。

3. 建向量列
```sql
ALTER TABLE memories ADD COLUMN embedding BLOB;
```
固定 embedding 模型，我用本地 bge-small-zh，维度 512。写入时保存 `struct.pack('<512f', *vec)`，避免 float32 和 float64 混用。

4. 召回流程
- 先按 `scope` 过滤，缩小候选；
- FTS 取 top 20，用 BM25 rank；
- 向量取 top 20，余弦相似度；
- RRF 融合：`score = 1/(k + rank_fts) + 1/(k + rank_semantic)`，k 取 60 即可。

工具返回结构：
```json
{
  "items": [
    {"id": 12, "content": "...", "source": "fts", "score": 0.032},
    {"id": 7,  "content": "...", "source": "semantic", "score": 0.91}
  ]
}
```
让 Agent 根据 source 判断这条记忆是因为“关键词命中”还是“语义相近”回来的。

## 踩坑点
1. **纯语义误召回比漏召回更伤**。Agent 拿到不相关内容会把它当上下文，产生幻觉。工程域记忆里，宁可少给，不要给错。
2. **中文 FTS 默认分词不行**。unicode61 按空格切，中文整句变成一个 token。必须上 trigram 或外部分词，否则中文全文搜索基本不可用。
3. **embedding 模型版本和 prompt 格式必须固定**。bge 系列 query 侧要加前缀，chunk 侧不加。模型升级后旧向量必须重算，不能混查。
4. **阈值不要拍脑袋**。不同 query 的余弦相似度分布差异很大。0.7 对某些 query 太严，对某些太松。先记录一批真实 query，标定 top-k 和相似度分布，再写死阈值。
5. **元数据过滤要放在召回前**。先 `WHERE scope=?` 再算向量，不要全局 top 后过滤，否则会漏掉当前 scope 的优质结果。

## 可复用建议
- 记忆量小于 1 万条，先只上 SQLite FTS5 + 元数据过滤，足够覆盖大部分工程召回。
- 只有当“同义改写”类需求频繁出现时，再加语义搜索，用 RRF 混合，不要用语义结果替代 FTS。
- 给 memory_recall 工具加 `mode` 参数，便于调试回归。
- 记录 recall 日志：query、mode、top items、是否被 Agent 引用。离线评估再决定要不要调 RRF 权重。
- 如果 embedding 检索用 sqlite-vec，注意它返回的 distance metric，L2 和 cosine 不要混用。

## 总结
memory_recall 不是一个模型能力问题，而是一个信息检索工程问题。全文搜索解决“精确命中”，语义搜索解决“模糊相关”。在 OpenClaw 这类 Agent 场景里，先让 FTS 兜底，再用语义搜索补召回，配合 scope 过滤和固定 embedding 版本，比直接上向量库更可控。别把记忆系统做成“看起来高级但经常返回错误上下文”的装饰品。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/140a5180502b97ac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/8a5d0067b1648d72.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/59b36bc8870c32fc.png)

