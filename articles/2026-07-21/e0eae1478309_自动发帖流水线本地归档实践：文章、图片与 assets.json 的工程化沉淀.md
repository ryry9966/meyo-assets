---
title: 自动发帖流水线本地归档实践：文章、图片与 assets.json 的工程化沉淀
feedId: 29899
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在基于 OpenClaw / Agent 的自动发帖流水线中，内容生产的速度越来越快，但「内容资产的本地化沉淀」往往被忽视。常见场景是：Agent 调 MCP 工具生成文章、抓取配图，再通过发布插件推送到各个平台。整个过程顺畅，可一旦需要回溯某篇已发内容、复现某次生成效果、或对历史图片进行二次处理，就会发现本地只剩混乱的临时文件和空白的日志。

我们团队维护的自动化内容系统每天生成数十篇多平台帖子，最初也面临类似问题：文章正文散落在多个临时目录，图片文件名随机，发布完成后无法追溯当时的资源列表。后来我们参考静态站点生成器（如 Hugo）的内容组织方式，设计了一套基于 `assets.json` 的本地归档结构，让每一条生成内容都是可审计、可复用的静态资产包。

这篇文章将分享整个归档方案的设计、实施步骤、踩过的坑，以及如何把它封装成可复用的 Agent 插件或脚本。

## 问题拆解

自动化发帖流水线通常会经历：选题 → 生成正文 → 生成/匹配图片 → 格式化（Markdown、富文本等） → 平台分发。多数工程师会关注生成质量和发布成功率，但以下问题往往被低估：

1. **内容碎片化**：文章和图片的对应关系只存在于内存或临时变量中，发布后散落或无记录。
2. **图片引用失效**：如果图片先上传到 CDN 再引用，本地没有原始副本，日后无法重新编辑或迁移。
3. **缺乏统一索引**：想统计“某段时间内生成了多少内容”“某张图被哪些文章使用过”，靠 grep 远远不够。
4. **调试困难**：出现问题后，需要重新跑一遍流水线才能复现上下文，代价很高。

因此，我们需要在每个发帖任务完成后，在本地留下一份“静态快照”，包含：文章正文（Markdown）、用到的所有图片的本地副本、一份描述元数据和资源映射的 `assets.json`。这样无论后续管道如何变化，这次发布的状态都被完整固化为一个目录。

## 归档目录结构设计

我们约定一次生成任务（即一篇帖子）对应一个独立目录，命名规则为 `YYYY-MM-DDTHH-mm-ss_<slug>`，例如 `2024-12-23T09-05-32_hello-world`。目录内部结构如下：

```
2024-12-23T09-05-32_hello-world/
├── index.md           # 文章正文（Markdown）
├── images/            # 所有配图的本地副本
│   ├── cover.png
│   └── figure-01.png
└── assets.json        # 文章元数据与资源清单
```

`assets.json` 肩负“内容清单”的角色，其基本 Schema 设计为：

```json
{
  "id": "2024-12-23T09-05-32_hello-world",
  "slug": "hello-world",
  "title": "Hello World",
  "created_at": "2024-12-23T09:05:32Z",
  "platforms": ["twitter", "blog"],
  "main_md_file": "index.md",
  "images": [
    {
      "file": "images/cover.png",
      "alt": "cover image",
      "remote_url": "https://cdn.example.com/2024/12/cover.png",
      "original_filename": "gen_cover_001.png",
      "width": 1200,
      "height": 630
    }
  ],
  "custom_meta": {
    "tags": ["automation", "openclaw"],
    "generated_by": "gpt-4o-2024-11-20"
  }
}
```

关键字段说明：
- `id`、`slug`、`title` — 基础标识；
- `platforms` — 记录已发布到的平台，便于后续回传或状态同步；
- `images` — 数组化记录每张图片的本地路径、远程 URL、原始文件名、尺寸等，做到了“图-文-源”全对应；
- `custom_meta` — 可扩展的生成参数，如大模型版本、温度、提示词摘要，供调试复用。

这样的目录一旦生成，就可以直接归档、压缩或同步到 Git 仓库，且与平台解耦。

## 实施步骤（基于 OpenClaw + Python 脚本）

我们把这个归档逻辑封装为一个独立的 Python 脚本 `archive_post.py`，由 Agent 在发布成功后调用。核心步骤如下：

### 1. 创建任务目录并写入 Markdown
从 Agent 上下文获得最终的文章内容（Markdown 字符串），以 `slug` 为基础生成时间戳目录，直接写入 `index.md`。注意 slug 需要做文件名净化（移除空格、特殊字符），使用 `slugify` 库即可。

### 2. 拷贝图片到 `images/`
遍历本次任务用到的所有图片本地路径（此时图片可能来自 MCP 工具返回的临时路径，如 `/tmp/gen_img_xxx.png`），分别拷贝到目录的 `images/` 下，并重命名为有语义的名字（如 `cover.png`、`figure-01.png`）。**保留原始文件名的字段**则记录在 `assets.json` 中，以便追溯源头。

### 3. 构建 `assets.json` 并写入
收集所有元信息后，序列化为带缩进的 JSON 写入。这里要注意 UTF-8 编码，并确保路径分隔符统一为 `/`。

### 4. Agent 集成方式
在 OpenClaw 的 Pipeline 定义中，增加一个 Post-Action 节点：

```yaml
post_actions:
  - name: local_archive
    plugin: python_script
    config:
      script: archive_post.py
      args:
        slug: "{{ post.slug }}"
        title: "{{ post.title }}"
        content: "{{ post.content }}"
        images: "{{ post.image_paths | json }}"
        platforms: "{{ post.platforms }}"
        meta: "{{ post.custom_meta }}"
```

这样每篇帖子发布后，立即生成归档目录，流水线无需感知额外细节。

## 踩坑点

1. **路径处理跨平台**：流水线可能在 Linux 容器中运行，但测试环境在 macOS。`pathlib` 是好朋友，但写入 `assets.json` 的路径要强制使用正斜杠，否则跨平台读取时会出错。
2. **同名图片覆盖**：如果一篇帖子使用了两张不同来源但都是 `cover.png` 的图片，拷贝到同一目录会冲突。我们改为在 `images/` 内添加子目录（如 `images/source-a/cover.png`）或在命名时加入哈希前缀，最终方案是统一命名为 `cover-<hash>.png`，并在 `assets.json` 中用 `alt` 区分语义。
3. **JSON 序列化循环引用**：在传入复杂自定义元数据时，可能包含不可 JSON 序列化的对象（如 datetime）。我们定义了 `CustomEncoder` 并在脚本中统一做安全序列化，避免因一个字段导致整个归档失败。
4. **磁盘占用**：随着发布量增加，图片副本会占用大量空间。对于纯存档需求，可以在归档后使用 `pngquant` 或 WebP 无损压缩，脚本中增加了可选的压缩步骤。或者定期将旧归档打包并移动到对象存储，本地保留最近一周的内容。
5. **幂等性**：若流水线因重试机制重复执行发布后动作，可能导致归档目录重复创建。我们在创建目录前检查 `assets.json` 是否存在，若存在则视为已归档，直接跳过，避免内容被覆盖。

## 可复用建议

- **与 MCP 图片工具结合**：如果图片由 MCP 图片生成服务（如 DALLE、Stable Diffusion）返回，归档脚本可以直接从 MCP 返回的 URL 下载图片到本地，而不依赖 CDN 持久化，这样即便 CDN 清理，本地依然有高清原图。
- **作为“内容回溯”的基础设施**：归档目录可直接作为静态站点源（如搭配 Hugo、11ty），批量重建历史内容，方便本地预览或生成静态镜像站。
- **扩展 `assets.json` 支持多语言**：如果同时生成了多语言版本，每个语言一个子目录，`assets.json` 增加 `locales` 字段记录映射。
- **集成到 OpenClaw 插件生态**：可将上述脚本封装为 `openclaw-plugin-archive`，提供标准配置项，社区用户只需在 pipeline 中引用即可获得本地归档能力。
- **版本控制友好**：目录名包含时间戳和 slug，Git 友好，便于通过 `git diff` 查看内容变更历史（比如修改了标题而重新发布）。

## 总结

自动发帖不应该只是“推出去”，还应是一次完整的资产沉淀。通过建立“每篇文章一个目录、一篇 `assets.json` 清单”的本地归档制度，我们得到了可追溯、可复用、跨平台中立的静态内容资产。这对后续的调试、审计、内容再加工以及训练数据集构建都极为有益。

在工程实践中，这个方案额外增加的磁盘操作耗时通常不超过 200 毫秒，对流水线性能影响微乎其微，但带来的长期维护收益却非常可观。如果你正在搭建或优化自己的 Auto-Post Agent，建议将这个模式纳入必选项，它会让你在任何需要“回过头看”的时候，都保持从容。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/25b7058339bc203b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/9f725a6a3694cc31.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/c900b2dff6e931c6.png)

