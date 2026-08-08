---
title: GitHub + jsDelivr 低成本图床实战：为你的 Agent 自动化管线配上稳定图片服务
feedId: 32188
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在自建 OpenClaw 工作流或基于 Agent 的自动化写作管线中，经常需要输出带图片的 Markdown。本地路径无法直接通过消息通道分发，远端对象存储又常常绑定云厂商、产生账单。如果只想为笔记、issue 或内部机器人配几张示意图，一个零成本、可版本管理、且自带全球 CDN 的图床方案是极其实用的组合：**GitHub 仓库 + jsDelivr**。

这篇文章不是图床选型清单，而是聚焦在实现上：如何在自动化管线里可靠地生成图片链接，并避免常见的缓存、文件大小、URL 编码等踩坑点。适合已经接触过 OpenClaw/Agent/MCP/插件开发，需要集成一个可控图片层的用户。

## 问题

在写一个自动化周报 Agent 时，我需要把本地生成的截图（如 Playwright 截屏、PlantUML 图、渲染数据看板）嵌入到最终输出的 Markdown。起步时用了本地文件路径，但发给即时通讯工具或发布到 GitHub Pages 时链接全部失效。尝试过几个免费图床，有的加了防盗链，有的悄无声息删图，稳定性堪忧。

我需要一个方案满足：

- **免费、无需信用卡**，个人项目零成本运转
- **全球可访问**，机器人和阅读者都在不同地区
- **支持 Git 工作流**，图片和代码一起版本控制
- **能被脚本/API 驱动**，方便集成到自动化管线

GitHub 仓库托管原始文件，jsDelivr 作为 CDN 加速，恰好命中这些点。

## 做法 / 步骤

### 1. 创建专用图床仓库

在 GitHub 新建一个公开仓库（private 仓库文件无法通过 jsDelivr 访问）。推荐命名如 `img-bed`，初始化 README。

仓库大小限制 1GB，单文件建议控制在 10MB 以内（jsDelivr 对超 20MB 文件可能不缓存）。如果你的管线产生的是截图，通常几十到几百 KB，完全够用。

### 2. 组织目录与文件命名

根据来源或项目分目录，例如：

```
/img-bed
  ├── screenshots/
  ├── diagrams/
  └── blog/
```

文件名避免空格、中文和特殊字符。使用 `20250101-login-flow.png` 这类可读格式，而不是 `image (1).png`。原因后面踩坑点会说。

### 3. 上传图片

**手工方式：** 直接拖拽到 GitHub 网页或 `git push`。之后通过 jsDelivr 的标准 URL 访问：

```
https://cdn.jsdelivr.net/gh/你的用户名/仓库名@版本号/文件路径
```

例如：`https://cdn.jsdelivr.net/gh/myname/img-bed@main/screenshots/20250101-login-flow.png`

使用分支名（如 `@main`）会自动指向最新 commit，但缓存刷新有延迟；后面会说明如何主动 purging。

**自动化脚本：** 在 Node.js 或 Python 管线中，借用 `@octokit/rest` 或 `PyGithub` 调用 GitHub API 上传。核心步骤：

1. 读取本地图片并转为 base64。
2. 调用 `PUT /repos/{owner}/{repo}/contents/{path}`，提交信息记录操作日志。
3. 组装 jsDelivr URL 返回。

一个最小 Node.js 示例（依赖 `@octokit/rest`）：

```js
import { Octokit } from "@octokit/rest";

const octokit = new Octokit({ auth: process.env.GH_TOKEN });
const content = fs.readFileSync("screenshot.png", { encoding: "base64" });

await octokit.repos.createOrUpdateFileContents({
  owner: "myname",
  repo: "img-bed",
  path: "screenshots/20250101-login-flow.png",
  message: "Add login flow screenshot [skip ci]",
  content,
  branch: "main"
});
```

上传成功后返回 `https://cdn.jsdelivr.net/gh/myname/img-bed@main/screenshots/20250101-login-flow.png` 供下游使用。

### 4. 集成到 Agent 管线

在 OpenClaw 或自定义 Agent 中，将上图操作封装成一个 MCP tool 或插件函数。当 Agent 需要生成图片链接时，调用此函数获得 CDN 地址，再嵌入 Markdown。整个过程无需退出上下文，也不必手动复制。

如果你在用 GitHub Actions 自动发布博客，可以写一个 Action 监听图片目录变更，自动把图片用 `git push` 同步到图床仓库，并用 `purge` 刷新缓存，使新图片即时生效。

## 踩坑点

### 缓存刷新不及时

jsDelivr 会缓存文件，更新同一路径下的图片后，CDN 边缘节点可能仍返回旧内容。**不要依赖 `@latest` 或分支名等待其自动刷新**，需要主动触发：

访问 `https://purge.jsdelivr.net/gh/用户名/仓库名@分支名/文件路径`，即时清除缓存。可以将此操作写进上传脚本的尾部。

```bash
curl -X PURGE https://purge.jsdelivr.net/gh/myname/img-bed@main/screenshots/20250101-login-flow.png
```

如果使用版本 tag 而不是分支，可以避免此问题，每次新图片使用新 tag，但管理成本略高。个人项目用 purge 足够。

### 文件名中的特殊字符

空格、中文、`#`、`?` 等字符会导致 URL 编码不一致，部分客户端无法正确解析。务必在脚本中对文件路径做 encodeURIComponent，或直接限 ASCII 字符、下划线、短横线。上传之前检验文件名字符集。

### 仓库大小与滥用限制

GitHub 建议仓库小于 1GB，超过会邮件警告。虽然不会直接封停，但应避免将图床当做大容量存储。定期清理历史图片，或对超大文件用 Git LFS（但 LFS 带宽有额外计费）。图片建议使用有损压缩，PNG 可转为 WebP 再上传。

### 公开仓库的隐私问题

所有图片均可被任何知道链接的人访问。别上传包含敏感信息的截图。若需要私有图床，可考虑 Cloudflare R2 等，但那不是本文的零成本范围。

## 可复用建议

我把上述上传和刷新逻辑封装成一个轻量 NPM 包 `imgcdn-kit`（可自行实现），供所有 Agent 项目调用。在设计工具接口时，遵循以下原则：

- 输入本地图片路径（或 Buffer），输出完整 CDN URL。
- 自动处理文件名规范化、重复命名（添加时间戳）。
- 支持 dry-run 模式，仅验证连接不真实上传。
- 返回的结构包含原始尺寸、URL、markdown 片段，方便拼接。

同时，本地 debug 时可以使用代理（例如 Whistle）将 jsDelivr 域名映射到本地 HTTP 服务，避免频繁上传浪费配额。

如果你的项目已经依赖 PicGo 等图形化工具，同样可以配置 GitHub 图床，但自动化场景下命令行/API 方式更灵活。

## 总结

GitHub + jsDelivr 胜在**确定性**：免费额度的边界清晰，行为可通过 API 精确控制，而且与版本管理天然结合。对于个人 Agent、小团队自动化、开源项目文档图床，它足够好。它不是高并发、大流量场景的选择，但对于大多数自动化管线——每天几十次上传，偶尔几千次读取——完全撑得住。

配上一小段上传脚本和一个 purge curl，你就能为自己的 OpenClaw 工作流添加一条固态的图片供给线。

---

