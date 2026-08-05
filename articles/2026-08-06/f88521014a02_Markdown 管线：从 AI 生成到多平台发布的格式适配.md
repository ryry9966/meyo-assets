---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 31802
source: 综合讨论
publishedAt: 2026-08-06
---

# Markdown 管线：从 AI 生成到多平台发布的格式适配

## 背景

用 Agent 或自动化脚本产出 Markdown 已经是很多 OpenClaw 用户的日常。无论是让大模型直接输出文章，还是通过 MCP 服务抓取信息后拼装成文档，Markdown 作为中间格式具备了简洁、可版本控制、易于转换的优点。但问题往往出现在“最后一公里”——你需要把同一份 Markdown 发到公众号、知乎、Notion、飞书甚至邮件，而每个平台对格式的理解千差万别：公众号需要一个自带样式的 HTML 片段，知乎可以识别 Markdown 但公式渲染容易出 bug，Notion 导入时图片不会自动上传，飞书云文档则需要特定的块结构。

手工逐个平台调整格式不仅耗时，还容易引入不一致。于是我们搭建了一条“Markdown 管线”：从 AI 生成或人工撰写的源文件出发，经过校验、预处理、按目标平台组装，最终输出可直接发布的产物。

## 问题定义

核心挑战可以拆成三部分：

1. **源文件质量问题**：AI 直接输出的 Markdown 常有不规范之处，比如混用列表缩进、代码块未指定语言、链接缺少标题等，直接转换会放大错误。
2. **平台差异抽象**：公众号要求内联样式且禁止外链 JavaScript，知乎需要处理公式 `$$` 与 HTML 实体的冲突，Notion 需要先上传图片再嵌入，飞书则需要 token 鉴权调用文档 API。
3. **可维护性与自动化**：如果每次发布都要手工跑脚本，就容易退化回手动粘贴。需要与现有的 Agent 工作流或 Git 事件结合，做到无感触发。

## 做法与步骤

我们采用 **单源多目标** 思路，以标准 Markdown 文件（扩展 `.md`）为唯一手写或 AI 生成入口，目录结构如下：

```
content/
  source/
    my-post.md
    images/
      cover.png
  output/
    wechat.html
    zhihu.md
    notion.json
    feishu.json
tools/
  lint_md.py
  convert_wechat.py
  convert_zhihu.py
  upload_notion.py
  upload_feishu.py
Makefile
```

### 1. 统一质量卡控（Lint）

在 `tools/lint_md.py` 中，我们封装了 `markdownlint` 的调用并加入自己的规则，比如：

- 强制要求一级标题只有一个（`MD002`）
- 代码块必须指定语言（`MD040`）
- 禁止在段落中使用 HTML 表格（平台兼容性差）
- 检查所有本地图片引用路径是否存在

这一步作为管线的第一关，任何 AI 生成的内容在进入转换前都会先被 lint。我们在 Makefile 里定义：

```
lint:
    python tools/lint_md.py content/source/*.md
```

所有后续步骤都依赖 `lint` 通过。

### 2. 预处理与模板

在转换前对源文件做一次规范化：用 Python 的 `markdown` 库解析 AST，修复常见问题。例如，将 `![](./images/xxx)` 统一为相对根路径，给图片添加 `alt` 文本（如果缺失则用文件名填充）。同时，利用 frontmatter 提取标题、摘要、标签，供不同平台模板使用。

### 3. 按平台生成产出物

**公众号（`convert_wechat.py`）**：  
使用 `pandoc` 将 Markdown 转为 HTML，然后通过自定义的 `--lua-filter` 做几件事：  
- 将所有 `<a>` 标签的 `href` 替换为微信可识别的外链格式（需要已备案域名），对不可外链的链接转为脚注文本。  
- 图片加上 `max-width: 100%` 的内联样式，并去掉 `figure` 包裹。  
- 代码块转为公众号可渲染的 `<pre><code>` 并内嵌简易高亮样式。  
- 生成的 HTML 片段可以直接粘贴到公众号编辑器。

**知乎（`convert_zhihu.py`）**：  
保留 Markdown 格式，但需要对公式块做特殊处理：将 `$$` 包裹的 LaTeX 转义为知乎可接受的 `![公式](https://www.zhihu.com/equation?tex=...)` 。同时，图片需要用知乎图床上传并替换链接，这一步我们复用了已有的 Cookie 上传脚本，但注意知乎的防盗链策略变化频繁，因此也支持回退为保留本地路径手动上传。

**Notion（`upload_notion.py`）**：  
用 Notion API 创建页面并追加块。核心痛点在于图片：API 不直接接受本地文件，必须先用 `files` 属性上传得到临时 URL，再嵌入 `image` 块。所以我们先扫描 Markdown 中的所有图片引用，批量上传并缓存映射表，再替换内容中的链接。块结构的顺序和嵌套也容易出错，建议用官方 SDK 的 `append_block_children` 逐步构建，避免一次性生成整个页面导致缩进层级混乱。

**飞书（`upload_feishu.py`）**：  
通过飞书云文档 API，将 Markdown 分段转换为飞书的 `blocks` 格式。这里需要处理表格、分割线等飞书不直接支持的块类型，转化为富文本描述或截图。最稳妥的方式是预先将复杂表格渲染为图片，以 `image` 块插入。

### 4. 串联与触发

所有转换脚本统一用 Makefile 管理：

```
all: wechat zhihu notion feishu

wechat: lint
    python tools/convert_wechat.py content/source/my-post.md > content/output/wechat.html

zhihu: lint
    python tools/convert_zhihu.py content/source/my-post.md > content/output/zhihu.md
    # 可选的上传图片步骤
...
```

进一步，可将这个管线封装成 MCP 服务，供 OpenClaw 中的 Agent 直接调用：Agent 生成 Markdown 后调用 `convert` 工具，选择目标平台，内部走完 lint 和转换，返回发布链接或可粘贴的 HTML。这样就把人工那一环也省掉了。

## 踩坑点

1. **Pandoc 数学公式转换的坑**：转 HTML 时默认会将 `$...$` 包裹在 `<math>` 或 `\(...\)` 中，公众号和多数富文本编辑器不识别。我们改用 `--mathjax` 并配合 filter 将所有公式替换为渲染好的 SVG 图片，但代价是构建时间变长。
2. **微信外链与防盗链**：公众号正文中的链接必须是指定域名，且不能是短链。必须维护一份域名白名单，并在 filter 中自动归类外链，将不合规的链接转为纯文本提示。
3. **图片路径与缓存**：多平台图片上传很容易触发 API 限流或失效。建议在预处理阶段就统一对所有本地图片生成 MD5 哈希，缓存上传结果，避免重复上传。
4. **飞书块嵌套限制**：飞书的无序列表嵌套最多三级，超过会报错。需要在转换时将过深的嵌套展平或转为引用块。

## 可复用建议

- **强制 frontmatter 规范**：所有 Markdown 源文件必须包含 YAML frontmatter，至少声明 `title`、`description` 和 `tags`，这样各平台模板能直接拿元数据填充摘要和 SEO。
- **用 CI 做守门员**：在 Git 仓库中配置 pre-push hook 或 GitHub Actions，每次 push 都自动 `make lint`，不通过则拒绝合并，确保 AI 生成的 PR 也不会夹带格式问题。
- **抽象一个 Platform Adapter**：不要把平台转换逻辑散落在脚本里。可以定义一个 Python 抽象基类 `PlatformAdapter`，包含 `upload_image`、`convert_markdown`、`create_document` 等方法，每个平台继承实现。这样新增平台时只需关注差异部分。
- **管线即 MCP 工具**：如果已经大量使用 Agent 驱动内容生成，直接把整条管线暴露为 MCP tool，Agent 在生成完内容后可以直接调用 `publish_to_wechat` 完成发布，大大减少人工操作。

## 总结

Markdown 管线并不是银弹，它解决的是“同一份内容多渠道分发”过程中的重复劳动和格式漂移问题。对 OpenClaw 用户而言，把 AI 生成、排版校验、平台适配串联成一条可自动化执行的流程，既能保持内容产出的敏捷，也能避免发布时的手忙脚乱。建议从小场景开始，比如先打通“MD→公众号 HTML”这一条线，再逐步扩展到其他平台，过程中积累的 adapter 和 lint 规则会成为团队可复用的基础设施。

---

