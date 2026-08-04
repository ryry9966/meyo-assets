---
title: GitHub + jsDelivr 搭建免费图床的实践：Agent 工作流的图片外链方案
feedId: 31624
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

在 OpenClaw 的自动化工作流里，经常遇到一个共性需求：Agent 生成了图表、渲染结果或分析截图，需要给到一个外部系统可以直接访问的 URL，用于展示、存档或者被另一个 Agent 消费。

最常见的做法是用对象存储（R2、OSS、S3），但每次都要处理凭证、权限桶、生命周期策略，对非专业的内部工具来说成本并不低。而用免费的公开图床，又面临链接失效、防盗链、删除策略等不可控因素。

这篇笔记记录我目前认为最务实的一套组合：GitHub 仓库作为图床本体，jsDelivr 作为 CDN 缓存层。不需要购买服务器，不需要维护域名。

## 问题

免费图床的核心矛盾在于三个点：

1. **稳定性**：免费图床的上传策略、审查策略随时可能变化，链接不可长期承诺。
2. **缓存**：大部分图床对更新后的同名文件有缓存失效滞后，对于 Agent 自动化场景非常致命——你生成的图表在别人打开时还是旧版本。
3. **可控性**：能否通过脚本批量上传、删除、列出仓库里的对象？

GitHub + jsDelivr 在这三点上恰好都满足：jsDelivr 是显式 CDN，带有可调用的 purge 接口；GitHub 是代码托管平台，仓库的公开访问是长期稳定的；repo 的文件系统天然支持 API 操作。

## 做法 / 步骤

**第一步：建立专用图床仓库**

新建一个公开仓库 `assets`，路径规范为：

```
assets/{project}/{date}/{filename}.png
```

用日期分层的好处是：后续做归档清理时，可以直接脚本遍历 `date` 前缀，按时间批量降级或删除。

**第二步：推送文件**

用 GitHub CLI 就行，不需要写死 token：

```bash
gh auth login
gh repo clone {yourname}/assets
cp output.png assets/2025/06/15/
git add -A
git commit -m "chore: add agent output"
git push origin main
```

如果想在 Agent 里直接调用，可以把这一步封装成一个 shell 脚本 `upload_to_cdn.sh`，接收文件路径和目标相对路径，内部完成 push。

**第三步：通过 jsDelivr 访问**

```
https://cdn.jsdelivr.net/gh/{yourname}/assets@main/2025/06/15/output.png
```

注意分支写法：`@main` 对应默认分支。jsDelivr 对 GitHub 仓库的缓存 TTL 是 12 小时，默认情况下基本够用。

**第四步：缓存失效**

需要即时刷新时，调用官方 purge 接口：

```bash
curl -X POST "https://purge.jsdelivr.net/gh/{yourname}/assets@main/2025/06/15/output.png"
```

在 Agent 工作流里，我通常建议把 `upload` 和 `purge` 放在同一个函数里，避免旧缓存造成排障假象。

## 踩坑点

- **仓库大小限制**：GitHub 仓库推荐不超过 1GB，如果 Agent 长期产生的图片没有生命周期，仓库会逐渐膨胀。建议在 Git LFS 与裸仓库之间做取舍——实际上 jsDelivr 不代理 LFS 文件，所以大文件不适合走这条路。我的建议是限制单文件 10MB 以内，超过的走对象存储。
- **分支大小写**：jsDelivr 的 URL 里 `@main` 必须与实际分支严格一致，维护多分支时容易踩坑。
- **公开可访问性**：如果图包含敏感信息，不要放到这个仓库，GitHub 仓库删除后，jsDelivr 的缓存仍可能存活一段时间。
- **无交互认证**：`gh auth login` 在无交互环境下可以用 `gh auth login --with-token` 配合 GitHub Actions Secret 来避免人工操作。
- **中文文件名**：会直接导致 URL 编码问题，建议脚本统一重命名为 ASCII 安全文件名（例如用 UUID 或时间戳哈希）。

## 可复用建议

1. 用 GitHub Actions 做定时清理：每天凌晨执行脚本，删除超过 90 天的目录，保持仓库体积可控。
2. 对 Agent 输出统一做压缩：截图类先转 WebP，再上传，体积能降 60%-80%。
3. 把 URL 拼接规则沉淀为一个通用函数，例如 `cdn_url(project, filename)`，这样下游 Agent 不需要关心仓库结构。
4. 如果你有多个环境（dev/prod），建议给 jsDelivr URL 加一层环境前缀。

如果你正在用 OpenClaw 跑 Agent，并且需要把这些图片结果回传给外部系统，这套方案实现成本很低，且在长期运行中几乎没有维护费用。

## 总结

GitHub + jsDelivr 本质上是用「公开仓库的可靠性」换「免费 CDN 的稳定性」。它适合：

- Agent 工作流的中间产物
- 不需要长期留存的可替换产物
- 对缓存敏感度要求不高的场景

不适合生产级用户内容分发，也不适合大文件形态。但如果你在跑个人的 Agent 自动化，需要快速、免费、可缓存失效的图床方案，这套组合是目前最稳的。

---

