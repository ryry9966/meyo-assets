---
title: 为自动发帖流水线加上本地归档：文章、图片与 assets.json 的落地方案
feedId: 30142
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在基于 OpenClaw 搭建的自动发帖流水线中，Agent 会调用 MCP 工具生成文章正文和配图，再推送到目标平台。几次跑下来，本地只留下一堆临时文件，查历史内容全靠打开平台后台，出问题时极难追溯。更尴尬的是，当你想把某篇生成质量不错的内容喂给“写作风格分析”工具时，发现原始 markdown 已经被删除，配图也散落各处。

这让作者意识到：流水线必须有本地归档——不依赖任何在线平台，把所有产出物以可控、可查询的形式固化下来。

## 问题拆解

一次典型发布流程中，我们会得到：

- **文章正文**：markdown 格式，可能包含绝对路径引用的图片链接
- **图片**：通过 DALL·E/Stable Diffusion 等工具生成，通常以临时 UUID 命名
- **元数据**：标题、标签、发布时间、生成参数等

归档要解决三个核心问题：

1. 如何确定归档目录结构，避免命名冲突且方便日后检索
2. 图片文件如何处理，路径引用怎么写才能既在本地可预览，又能在发布时转为平台 URL
3. 如何存储元数据，保证它既能被人类阅读，也能被程序批量消费

## 做法与步骤

### 1. 目录结构约定

我们采用 `年-月/日-摘要` 的层级，摘取自标题的前几个词做 slug，避免中文字符出现在路径中：

```
archive/
  2025-01/
    13-the-future-of-ai-agents/
      article.md
      images/
        cover.png
        diagram.png
      assets.json
    15-debugging-mcp-tools/
      article.md
      images/
        ...
      assets.json
```

这样按月份分桶，每个帖子独立目录，所有资源内聚，用 Git 管理时也干净。

### 2. 归档脚本的职责

在发帖流水线的最后一步（通常是“发布成功”的回调之后），触发一个脚本或 MCP 工具，完成：

- 从生成阶段的输出中提取文章标题、正文、本地图片路径列表、标签、生成模型等
- 创建上述目录结构
- 将正文中的图片路径替换为相对路径 `images/xxx.png`，方便本地阅读器预览，同时保留原图链接作为 `assets.json` 的原始 URL 字段
- 移动图片文件到 `images/` 下，并保留原始文件名（若冲突则追加序号）
- 生成 `assets.json`，记录关键元数据

### 3. assets.json 的设计

我们并不需要复杂 schema，够用、易扩展即可。实际使用的一份最小定义如下：

```json
{
  "title": "The Future of AI Agents",
  "slug": "13-the-future-of-ai-agents",
  "created": "2025-01-13T10:30:00Z",
  "published": "2025-01-13T11:00:00Z",
  "tags": ["ai-agent", "automation"],
  "platform_url": "https://example.com/posts/12345",
  "model": {
    "text": "gpt-4o",
    "image": "dall-e-3"
  },
  "images": [
    {
      "local_path": "images/cover.png",
      "original_url": "https://.../s3-xxx.png",
      "prompt": "A futuristic cityscape..."
    },
    {
      "local_path": "images/diagram.png",
      "original_url": "https://.../s3-yyy.png",
      "prompt": "A flow chart showing..."
    }
  ],
  "content_file": "article.md"
}
```

`original_url` 在归档时从生成工具的输出中获取，若不关心可直接留空。当需要重新发布到其他平台时，可读取 `assets.json` 将本地图片批注回线上 URL，做到可逆。

## 踩坑点

### 文件名与路径
Windows 下路径分隔符和长度限制是个常见坑。图片生成工具通常给出一长串 UUID 加随机后缀，容易超过 260 字符。统一在归档时使用简短命名（如 `cover.png`、`fig1.png`），原始文件名写入 `assets.json` 的备注字段即可。

### 图片引用替换的时机
正文 markdown 在归档前可能已经嵌入了线上 URL（若平台返回了上传后的地址）。脚本务必在发布**之前**保留一份原始本地引用副本，或借助 Agent 的上下文记录，否则归档到本地后图片引用全部是失效的外链。我们通过在内容生成阶段就输出“本地 markdown”和“发布 markdown”两份变量来解决。

### assets.json 的同步更新
多人维护或长流水线中，手动编辑 assets.json 极易造成与目录实际内容不同步。归档逻辑应作为唯一写入口，所有元数据由代码生成，禁止后续手动修改（除非在 Git 记录下修改）。

### 大图片与仓库膨胀
如果每天生成大量高清图，Git 仓库会迅速膨胀。建议对归档仓库使用 Git LFS 或定期将历史月份打包为 tarball 后归档到对象存储，`assets.json` 中增加 `archive_url` 字段指向离线包。

## 可复用建议

- **命名约定优于灵活**：统一 slug 生成规则（英文、小写、连字符），不要因“快速”而在路径中塞中文或空格。
- **assets.json 作为唯一真实来源**：任何下游工具（分析脚本、重新发布、数据统计）都不应猜文件名，而是读取 `assets.json`。
- **将归档纳入流水线测试**：在开发环境运行全流程，校验归档目录、图片可访问性、JSON 合法性。
- **考虑“冷存储”**：超过 6 个月的归档目录，定期从工作区移除，仅保留索引文件，释放本地空间。

## 总结

本地归档不是“备份思维”，而是让内容资产可被机器持续消费的基础设施。通过一个简洁的目录结构、一张 assets.json 元数据表以及少量脚本，就能让自动发帖流水线从“一次性快消”转变为可积累的知识工程。对于 OpenClaw 用户而言，这套方案非常轻量，不与任何平台绑定，结合 Git 还能获得完整的变更历史。希望你的每一篇自动生成内容，都有属于自己的归档小窝。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/e921ae33d8ba7bb6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/6031a4c7ed82f1c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/0c29077d1ff24995.png)

