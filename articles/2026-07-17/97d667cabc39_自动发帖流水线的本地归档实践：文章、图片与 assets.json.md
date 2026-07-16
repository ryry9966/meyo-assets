---
title: 自动发帖流水线的本地归档实践：文章、图片与 assets.json
feedId: 29340
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

自动化发帖流水线（例如通过 OpenClaw、MCP 或自建 Agent 定时生成并发布内容）让运营变得轻量，但也带来一个新问题：生成的文章和图片散落在临时目录、远程存储或平台缓存中。一旦流水线升级、容器重启或第三方服务失效，已有资产可能难以追溯，更谈不上复用。

本地归档不是「备份」，而是让每一条产出有可索引的结构化副本。本贴给出一种轻量、可工程化的做法，适用于 OpenClaw 生态中的自动化任务。

## 问题

- 发文 Agent 产出的 Markdown 原文和配图，怎么有组织地落盘？
- 如何建立元数据索引，方便后续搜索、重发布或数据集构建？
- 归档逻辑如何嵌入现有流水线，不增加过多耦合？

## 做法

在自动发帖流水线的末端插入一个**归档动作**，完成三件事：
1. 将文章正文存为 Markdown 文件；
2. 将文章中引用的图片下载到本地；
3. 生成标准化元数据文件 `assets.json`，记录文章、图片和提示词等信息。

### 目录结构

采用按日 + slug 分组的目录树：

```
archive/
  2025-01/
    2025-01-17-why-agent-needs-local-cache/
      2025-01-17-why-agent-needs-local-cache.md
      images/
        cover.png
        diagram.png
      assets.json
```

文章文件名与目录名一致，可读性高，也方便全局搜索工具索引。

### assets.json 设计

```json
{
  "slug": "2025-01-17-why-agent-needs-local-cache",
  "title": "Why Agent Needs Local Cache",
  "created_at": "2025-01-17T10:30:00Z",
  "platform": "xiaohongshu",
  "tags": ["agent", "engineering"],
  "images": [
    {
      "file": "images/cover.png",
      "remote_url": "https://cdn.example.com/img/cover.png",
      "alt": "架构示意图",
      "role": "cover"
    }
  ],
  "prompt_meta": {
    "system_prompt_hash": "abc123",
    "user_prompt": "写一篇关于...",
    "model": "claude-3.5-sonnet"
  },
  "archive_version": "1.0"
}
```

关键字段解释：
- `images` 中保留 `remote_url`，方便回溯原始发布引用，同时记录本地相对路径；
- `prompt_meta` 保存生成时的提示词摘要，后期可用来审计生成质量或复现。

### 集成到 OpenClaw 流水线

如果已经在用 OpenClaw 的任务编排，可以通过以下两种方式实现归档：

**方案 A：调用 MCP 文件系统工具**
- 使用 `@modelcontextprotocol/server-filesystem` 暴露归档目录；
- 在流水线步骤中，直接调用该 MCP 工具完成文件写入、图片下载（需额外辅助工具处理下载）。

**方案 B：使用 Shell 工具调用归档脚本**
- 编写一个轻量脚本 `archive.sh` 或 Node.js 脚本，接受标题、正文、图片列表作为参数；
- 通过 OpenClaw 的 `exec` 能力触发归档。

以方案 B 的 Node 脚本片段为例：

```javascript
// archive.js 接收 JSON 参数
const { title, slug, body, images, prompt } = JSON.parse(process.argv[2]);
const dir = path.join(ARCHIVE_ROOT, getDateDir(), slug);
fs.mkdirSync(dir, { recursive: true });
fs.writeFileSync(path.join(dir, `${slug}.md`), body);

// 下载图片并写入 images/ 目录
for (const img of images) {
  const filename = path.basename(new URL(img.url).pathname);
  const dest = path.join(dir, 'images', filename);
  downloadSync(img.url, dest);
  img.local_path = path.relative(dir, dest);
}

const assets = {
  slug, title,
  created_at: new Date().toISOString(),
  images: images.map(i => ({ ...i, local_path: i.local_path })),
  prompt_meta: { user_prompt: prompt }
};
fs.writeFileSync(path.join(dir, 'assets.json'), JSON.stringify(assets, null, 2));
```

在 OpenClaw 的任务定义中，末尾加入：
```yaml
 - name: archive
   action: exec
   command: node archive.js '{"title": "$post_title", ...}'
```

## 踩坑点

**图片引用一致性**  
Markdown 正文中的图片链接是远程 CDN 地址。归档后，若希望本地 Markdown 能直接查看图片，需要把链接替换为本地相对路径。建议保留原始正文不变，在 `assets.json` 中维护远程与本地映射，避免污染发布用的原文。

**并发写入冲突**  
如果流水线同时处理多篇文章，可能有多进程写入同一 `assets.json`。由于我们按文章独立目录存 `assets.json`，不存在并发写同一文件的问题；只有写入总索引时会冲突，建议单独维护一个总索引或依靠目录结构按需检索。

**图片下载失败**  
外网抖动可能导致下载失败。脚本应增加重试（3次，指数退避），失败后将该图片记录为 `"status": "failed"`，不中断主流程。

**命名冲突**  
同一天同一 slug 重新生成，会覆盖旧归档。可在目录名中追加时间戳或版本尾缀，例如 `2025-01-17-slug-v2`，由流水线变量控制。

## 可复用建议

- 把归档脚本打包成可参数化的 MCP 工具，通过环境变量指定归档根目录，方便不同团队复用。
- 为 `assets.json` 定义 JSON Schema，放入团队代码仓库，并做 CI 校验。
- 定期将整个 `archive/` 目录纳入 Git 版本控制，相当于免费得到时间线回溯能力。
- 在 OpenClaw 中，将归档步骤抽成任务模板，新流水线直接引用。
- 结合 `grep` 或 `ripgrep` 对归档目录做全文检索，快速找到历史内容。

## 总结

本地归档看似独立，却是一套自动化内容系统能否长期健康运作的腰眼模块。通过约定优于配置的目录结构、轻量的 `assets.json` 元数据、以及与 OpenClaw/MCP 体系的低耦合集成，可以在不增加运维负担的前提下，获得可靠的内容资产管理能力。

这套方法同样适用于 Agent 生成的报告、分析结果、代码片段等任何需要结构化留存的产物。

---

