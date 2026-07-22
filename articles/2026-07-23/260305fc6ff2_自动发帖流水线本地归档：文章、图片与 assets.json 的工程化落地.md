---
title: 自动发帖流水线本地归档：文章、图片与 assets.json 的工程化落地
feedId: 30132
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

自动发帖流水线通常关注“生成与发布”两个环节，本地归档往往被视为可选项。但在实际工程中，只要你想把内容复用到多个平台、需要做质量回溯或简单版本管理，本地持久化就会变成刚需。尤其在基于 OpenClaw 构建的自动化写作场景里，Agent 可能调用 MCP 图文生成服务，产出 markdown 正文与若干配图。如果只推送远程，事后查找、替换或二次编辑会非常痛苦。

本文面向正在搭建自动发帖流水线的工程师，介绍一套轻量归档方案：将文章、图片和 `assets.json` 一起落盘，实现结构化、可重放的本地沉淀。

## 问题拆解

一次典型的自动写作流程会输出三样东西：

1. **文章正文**（通常是 markdown）
2. **配图**（可能是生成图片、截图或抓取的网络资源）
3. **元数据**（标题、标签、生成时间、图片来源、prompt 等）

归档要解决的四个关键点：

- **结构一致性**：文章引用的图片路径在本地可解析；
- **可追溯性**：知道每张图的原始来源，生成失败时能定位；
- **可重放性**：保留足够的上下文，随时能重新执行类似生成任务；
- **不侵入流水线**：归档动作尽量无感，封装成工具或 MCP server。

## 做法与步骤

### 1. 定义归档目录规范

推荐以日期或主题为维度组织目录：

```
posts/
└── 2025-03-16-hello-openclaw/
    ├── post.md
    ├── assets/
    │   ├── cover.png
    │   ├── diagram.png
    │   └── ...
    └── assets.json
```

如果流水线是事件驱动（如监控触发），也可以用 uuid 或 slug 命名。**核心原则：每个归档单元自包含，复制即迁移。**

### 2. 保存文章并修正图片引用

Agent 或脚本生成 markdown 后，不要直接落盘。需要做三件事：

- 将 markdown 中的远程图片 URL 全部下载到本地 `assets/` 目录；
- 将正文内的图片引用改为相对路径 `./assets/filename.png`；
- 在文件头插入 YAML front matter，写入标题、tags、生成时间等。

示例 `post.md`：

```markdown
---
title: OpenClaw 自动归档方案
tags: [automation, openclaw, mcp]
created: 2025-03-16T10:30:00Z
source_prompt: "写一篇归档方案的技术帖"
---
正文内容...
![架构图](./assets/diagram.png)
```

这一步可以用 Python 脚本 + `markdown-it-py` 解析 ast 完成，也可以用正则快速替换，但要注意异常处理（见踩坑点）。

### 3. 构建 assets.json

`assets.json` 是一份照片清单，记录每张图片的关键信息，方便后续检索、重新生成或审计：

```json
{
  "assets": [
    {
      "local_path": "assets/cover.png",
      "original_url": "https://image.cdn.xxx/generated/abc123.png",
      "prompt": "a tech knowledge card illustration, clean background",
      "size_bytes": 245120,
      "width": 1200,
      "height": 630,
      "mime_type": "image/png",
      "downloaded_at": "2025-03-16T10:32:00Z"
    }
  ]
}
```

为什么手写而不是直接依赖 front matter？因为：

- 文章主体不应塞进过多结构数据；
- `assets.json` 可以单独被工具链消费，例如批量重新生成模糊图片、同步到图床、构建索引等；
- 易于和 OpenClaw 的 memory 或知识库集成。

### 4. 集成到流水线

理想方式是将归档逻辑封装为 MCP server 或 Agent 可调用的工具函数。我在 OpenClaw 工作流里使用了一个简单的归档工具，接口设计如下：

```python
def archive_post(
    post_md: str,
    assets_info: list[dict],
    output_dir: str
) -> str:
    # 1. 创建目录
    # 2. 下载图片
    # 3. 替换引用
    # 4. 写入 front matter
    # 5. 写入 post.md 与 assets.json
    # 6. 返回归档路径
```

在 Agent 侧只需调用一次，返回归档路径，之后的任何环节（发布、通知、日志）都可引用该路径。如果需要留痕，可以直接提交 git。

## 踩坑与排障

### 图片下载失败如何处理

生成平台偶尔返回临时链接或过期链接，下载时收到 404。如果直接抛异常，整个归档会中断。做法是对每张图设置重试次数（例如 3 次），仍失败则使用占位图并在 `assets.json` 中标记 `"status": "missing"`，同时正文里保留原始 URL 加注释，便于后续人工补全。

### markdown 引用替换的边界情况

用正则匹配 `![alt](url)` 容易遗漏 title 或引用风格（如 `<img>` 标签），而且图片 URL 可能带有查询参数、包含括号。建议用 markdown 解析器获得 token 流，只处理 `image` 节点，可靠得多。同时注意本地文件名去冲突，可用 URL 的 sha256 前 8 位作为文件名。

### 大尺寸图片和存储膨胀

如果图片数量大或分辨率高，单个归档目录轻松超过 50MB。可以增加压缩步骤：对 PNG 转 JPEG（若不需透明通道）并限定最长边 1200px，完成后更新 `assets.json` 中的 `width/height/size_bytes`。另外定期清理数月前的图片或移到冷存储。

### assets.json 与 post.md 数据不一致

最容易出现的问题是：替换了图片引用但忘记更新 json，或者 json 里记录了文件但实际未下载成功。写好校验函数，归档完成后立即比较 `assets.json` 中的 `local_path` 与目录下实际文件列表，发现缺失立刻在 Agent 反馈中报警，必要时撤回发布流程。

## 可复用建议

- **封装成独立工具**：不依赖特定 Agent 框架，可作为命令行工具或 MCP server，供 OpenClaw、Dify、自定义脚本共用。
- **与 git 联动**：归档后自动 `git add && git commit`，message 用标题+时间戳，方便回溯内容变化。
- **留出一块“raw 目录”**：除了替换引用后的 `post.md`，也保存原始 markdown（raw_post.md），从里面可以找回未修改的远程链接，方便后续重新生成图片。
- **给 assets.json 加 schema**：用 JSON Schema 约束字段，接入流程前做校验，防止后续解析出错。
- **结合 OpenClaw 的 memory**：将 `assets.json` 的摘要写入知识库，当你用自然语言问“上次那张流程图的 prompt 是什么？”时，Agent 能快速回答。

## 总结

自动发帖流水线加上本地归档后，不再是一次性消耗品，而变成有历史、可演化、可迁移的数字资产。文章、图片与 `assets.json` 的组合看似简单，却解决了结构化、可追溯和可重放三个工程难题。关键不在于复杂的设计，而在于归档动作的可靠性和对异常的处理方式。

当你的 Agent 能自信地返回“文章已发布，归档在 posts/2025-03-16-hello-openclaw/”时，你会意识到：自动化闭环，从把成果踏实落盘开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/467323b94bf835c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/344101c65dbe0a62.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/26d58fedd10d02d7.png)

