---
title: 为你的自动发帖流水线加上本地归档：文章、图片与 assets.json 的工程化实践
feedId: 28900
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景

用 OpenClaw 搭建的自动发帖流水线多数已经能完成「选题 -> 生成文章 -> 生成配图 -> 多平台发布」的闭环。我自己的实践里，这些产物在发布后通常只保留在 Agent 的工作目录，或者被 MCP 工具使用后就被随手清理了。时间一长，当需要复盘某篇帖子的表现、回溯生成时的 prompt 效果，或者想把已发布内容二次加工成视频脚本、训练数据时，才发现根本没有完整的本地留档。

这种丢失不是数据真的没了，而是因为缺少一套堪用的本地归档机制——文章散落在临时目录，图片没有和文章关联在一起，想查某张封面图是用什么 prompt 生成的也无从找起。

## 问题

自动流水线在本地归档上的痛点主要有三：

1. **资产离散**：文章是 Markdown 文本，图片是独立文件，可能连文件名都是随机的，事后很难对应。
2. **元数据缺失**：没有记录生成这篇内容使用的模型、提示词、生成时间、发布目标平台，导致回溯困难。
3. **归档时机模糊**：如果归档步骤耦合在发布逻辑里，一旦发布流程改了，归档就跟不上；如果完全独立，又容易漏掉。

这些都不是大问题，但如果流水线每天产出几十条内容，半年后往回看就等于面对一堆没有索引的碎片。

## 做法与步骤

### 1. 定义统一的归档目录结构

对一个自动生成的内容产物，我最终定下的本地归档目录格式如下：

```
archive/
  2025-04-01-make-your-mcp-tools-idempotent/
    article.md
    images/
      cover.png
      img-1.png
    assets.json
```

- 文件夹命名：`日期-文章slug`。日期保证排序，slug 保证人眼可识别。
- `article.md`：生成后的最终 Markdown 正文（发布前状态）。
- `images/`：保留所有相关图片原图。
- `assets.json`：一份机器可读的资产清单。

### 2. 设计 assets.json 的 schema

`assets.json` 的核心是回答这几个问题：这篇文章是谁生成的、用什么参数生成的、包含哪些图片、每张图片的来源是什么。

经过几轮迭代，我沉淀的最小可行结构如下：

```json
{
  "meta": {
    "slug": "make-your-mcp-tools-idempotent",
    "title": "Make Your MCP Tools Idempotent",
    "created_at": "2025-04-01T10:30:00Z",
    "generated_by": "openclaw-agent-v1",
    "tags": ["MCP", "idempotency"],
    "publish_targets": ["openclaw-cn"]
  },
  "article": {
    "file": "article.md",
    "model": "gpt-4o",
    "prompt_version": "v3.2"
  },
  "images": [
    {
      "file": "images/cover.png",
      "prompt": "A clean architecture diagram ...",
      "model": "dall-e-3",
      "size": "1792x1024",
      "role": "cover"
    },
    {
      "file": "images/img-1.png",
      "prompt": "A troubleshooting flowchart ...",
      "model": "dall-e-3",
      "size": "1024x1024",
      "role": "illustration"
    }
  ]
}
```

设计上留了扩展余地：如果后续为文章生成了多语言版本，可以在 `meta` 里加 `locale`；如果用 Stable Diffusion 生图，可以加 `seed` 等字段。

### 3. 在流水线中集成归档步骤

我采用的做法是把归档做成一个独立的 MCP tool，而不是硬编码在 Agent 的 prompt 里。大致的调用顺序为：

1. Agent 调用内容生成工具，得到 Markdown 文本和图片 URL。
2. Agent 调用图片下载工具，把图片落到临时目录。
3. 所有产物准备完毕之后，Agent 调用 `archive_article` 这个 MCP tool，传入：
   - 文章标题、slug、tags
   - Markdown 内容
   - 图片文件路径及对应的元信息（prompt、role 等）
   - 发布目标
4. `archive_article` 工具内部完成：创建归档文件夹、移动或复制文件、写 `assets.json`、返回归档路径。

这种把归档逻辑封装成独立工具的好处是：无论上游的生成逻辑怎么变，只要最终能调用到这个工具，归档行为就能保持稳定，并且可以独立升级。

### 4. 善后：版本控制与定期检查

归档目录直接纳入 Git 管理（但忽略 images 里的大尺寸二进制文件，用 Git LFS 或单独存）。每次归档成功后做一次自动提交，提交信息带上 slug。这样再旧的内容，也可以通过 git log 找到对应的变更时间线。

另外单开了一个极简的 cron job，定期扫描归档目录：检查是否有 `assets.json` 缺失、图片路径指向的文件是否存在，然后把异常数上报到钉钉/飞书通知。这一步直接把“归档是否健康”从玄学变成了可观测。

## 踩坑点

几个在实践中容易忽略的地方：

- **slug 冲突**：不同 Agent 在不同线程里可能在同一天产出相同 slug 的文章。解决办法是在 slug 后追加微秒时间戳，例如 `make-your-mcp-tools-idempotent-20250401T103000Z`，或者由调用侧保证唯一性。
- **图片下载不稳定**：某次图片 URL 可能过期或网络抖动导致下载失败。如果直接中断归档，会导致已有内容也丢失。更好的做法是允许部分失败：`images` 数组里用 `status: "failed"` 标记，不阻断归档，事后统一重试。
- **并发写入 assets.json**：虽然我的流水线基本是顺序执行的，但如果用多个 Agent 并行处理不同内容，就不能简单用文件追加。改成每个归档目录完全独立（不共享同一个 json 文件），问题消失。
- **路径硬编码**：不同运行环境（容器 vs 本地）归档根目录不同。需要把 `ARCHIVE_ROOT` 做成环境变量，在 MCP tool 里读取，绝不在代码里写死路径。
- **归档时机过早**：如果在图片还没下载完成时就调用归档工具，得到的 images 列表不全。所以一定要在所有资产准备完毕、且校验通过之后再调用归档。

## 可复用建议

如果你也在维护类似的自动发帖流水线，下面是几条不绑定具体框架的建议：

1. **把归档做成独立 MCP tool**，而不是用自然语言让 Agent “帮我把这些保存起来”。工具比 prompt 更可靠。
2. **优先标准化 assets.json schema**，甚至可以在团队里出一份 schema 文档，让下游数据清洗、分析的同学直接消费这份 json。
3. **归档与发布彻底解耦**：归档失败不应该影响发布。工具内部捕获所有异常，写入日志，但不向上抛出。可以通过监控告警来发现问题。
4. **用 slug 做主键**，不要用数据库自增 ID。slug 是人类可读的，换机器、换数据库也能重放。
5. **小步迭代**：从最简单的“文章 + 图片拷贝到文件夹”开始，不要一上来就追求完美的元数据设计。跑通一个端到端流程，再逐步把 prompt、参数填进去。

## 总结

本地归档算不上什么高大上的架构，但对于高频产出的 AI 内容流水线来说，它相当于给整套系统上了保险。过去我们花大量精力调 prompt、优化排版、适配不同平台，但如果这些产物没有落盘归档，就是一次性消耗品。有了这套轻量的文章、图片与 assets.json 归档机制后，复盘生成质量、构建二次数据集、迁移发布平台都变得顺手得多。希望这个实践能给你的流水线补上最后一公里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/c8eec0a0a1d5db87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/29c80860f2c769ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/e0a1f67b6d71e3e4.png)

