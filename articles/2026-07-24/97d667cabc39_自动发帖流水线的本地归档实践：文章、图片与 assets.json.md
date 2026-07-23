---
title: 自动发帖流水线的本地归档实践：文章、图片与 assets.json
feedId: 30233
source: 综合讨论
publishedAt: 2026-07-24
---

# 自动发帖流水线的本地归档实践：文章、图片与 assets.json

## 背景
在用 Agent/MCP 构建内容发布流水线时，很多人的关注点集中在「生成」和「分发」——让模型写出文章、调好封面、推到平台。但流水线跑完之后，内容资产几乎完全留在平台侧，本地往往只剩一堆临时文件甚至什么都没有。一旦平台限流、误删或者要回溯某篇旧文重新编辑，就只能去翻远端草稿箱，甚至重新生成，造成成本和上下文损失。

自动归档不是为了“存个备份”这么简单，而是让内容生产进入可追溯、可复用的工程轨道。如果你已经通过 MCP 连接了文件系统，这件事可以把复杂度控制得很低。

## 问题
不归档的典型痛点：
- 重新生成同一主题浪费 token，行为不可复现
- 想复用旧图的 prompt 或素材找不到
- 审计需要人工截屏，非常脆弱
- 多平台发布后，各平台版本差异无法统一比对

因此需要一个轻量但结构化的本地归档方案，覆盖文章全文、所使用的图片文件，以及一条结构化的资产记录，供后续程序读取。

## 做法与步骤

### 1. 归档目录结构
约定一个固定的根目录，例如 `~/post-archive`，下面按平台和日期组织：
```
post-archive/
  zhihu/
    2025-06-01/
      post.md
      images/
        cover.png
        img-01.jpg
      assets.json
  xiaohongshu/
    2025-06-02/
      ...
```
选择日期粒度是因为大多数发布任务都以天为单位，且文件名不易冲突。

### 2. 在 Agent 流程中接入归档
假设你的流水线大致是：知识库检索 → 大纲生成 → 正文撰写 → 图片生成/选择 → 推送平台。归档步骤可以插在“推送平台”之后，或者并行执行。

使用 MCP 的 filesystem 服务（如 `@anthropic/mcp-server-filesystem`）或直接用 Node.js/Python 的 `fs` 模块都可以，关键是要在 Agent 的工具集中暴露以下能力：
- 创建目录（含递归）
- 写入文本文件（UTF-8）
- 下载图片到指定路径
- 读写 JSON 文件

在 Claude Desktop 或支持 MCP 的环境中，定义一个归档工具：
```json
{
  "name": "archive_post",
  "description": "Save generated post, images and metadata to local archive",
  "inputSchema": {
    "type": "object",
    "properties": {
      "platform": {"type": "string"},
      "date": {"type": "string"},
      "markdown": {"type": "string"},
      "images": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "url": {"type": "string"},
            "filename": {"type": "string"}
          }
        }
      }
    }
  }
}
```

实际执行时，Agent 调用该工具，工具负责：
1. 根据 platform/date 构造目录路径
2. 创建目录
3. 将 markdown 写入 `post.md`
4. 下载每个图片 URL，写入 `images/` 目录（文件名采用传入的 filename，或从 URL 提取并重命名避免冲突）
5. 构建 `assets.json` 并写入

### 3. assets.json 字段设计
以一次发布为单位，典型的 `assets.json`：
```json
{
  "post_id": "zhihu-2025-06-01-001",
  "title": "自动归档的文章标题",
  "created_at": "2025-06-01T10:30:00Z",
  "platform": "zhihu",
  "source_prompt": "写一篇关于...",
  "images": [
    {
      "filename": "cover.png",
      "alt": "架构对比图",
      "prompt": "a diagram comparing ...",
      "model": "dall-e-3"
    }
  ],
  "urls": {
    "published": "https://zhihu.com/...",
    "shortlink": ""
  },
  "metadata": {
    "word_count": 1200,
    "tags": ["自动化", "归档"]
  }
}
```
不用设计成通用标准，以“能回溯完整生成上下文”为目标即可。`source_prompt` 字段记录原始 instruction，非常实用。

### 4. 增量更新与幂等
如果同一天对同一平台发多篇，需要区分：可在 `post_id` 中加序号，或按小时分钟创建子目录。工具内部检查 `post.md` 是否已存在，若存在则跳过写入（或追加），图片同理。这可以防止因 Agent 重试导致的文件覆盖。简单实现即检查 `assets.json` 是否存在，存在则读取并合并新条目。

## 踩坑点
- **图片下载**：DALL·E/Midjourney 生成的 URL 可能带有效期，必须在生成后短时间内下载。建议在生成步骤完成后立即调用归档工具，不要缓存 URL 等晚些再下载。网络超时需要重试，可设 3 次重试、超时 15 秒。
- **文件编码**：在 Node.js 中指定 `utf-8`，Python 中指定 `encoding='utf-8'`，否则 Windows 下可能出乱码。
- **并发写入**：如果多个 Agent 实例同时跑，简单串行化就好，没必要上锁。单机流水线通常不会并发写同一文件。
- **assets.json 大文件**：如果积累成千上万条，单文件会很大，建议按月或按周分文件，比如 `assets-2025-06.json`。
- **路径安全**：用 `path.join` 构造路径，防止 `platform` 参数含 `../` 穿越目录。

## 可复用建议
可以把归档逻辑抽成一个独立的 MCP 工具插件，和发布工具解耦。在 Agent 指令中这样描述：“当文章已推送成功后，必须调用 archive_post 工具保存本地副本”。这样即使发布失败，也可能提前调一次归档保存生成物，便于排查。

另外，图片 prompt 和模型参数对回溯很重要，务必写入 `assets.json`。可视情况增加 `generation_config` 字段。

归档之后，本地文件可用于：
- 用 ripgrep/fzf 快速搜索旧文
- 用 python 脚本对比不同平台的同一篇文章
- 定期同步到私有 Git 仓库作为内容版本管理

## 总结
自动发帖流水线很容易变成“只管生不管养”，而一个几十行的归档工具就能把生成资产沉淀下来。工程上它不复杂，但价值会在你三个月后想重新改一篇旧文时明显体现出来。基于 MCP filesystem 实现归档，让 Agent 自身就具备“归档意识”，而不依赖外部云盘或复杂数据库，是保持轻量和可控的务实选择。

把文章、图片和 assets.json 同步落在本地，意味着你的内容在离开 Agent 内存之后，依然拥有结构化、可检索的生命力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/aafef085f3130bdd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/06dc23a240603683.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/eac5e5c1e537518e.png)

