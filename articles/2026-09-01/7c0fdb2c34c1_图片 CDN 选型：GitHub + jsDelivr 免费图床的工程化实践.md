---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的工程化实践
feedId: 35589
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw、Agent、MCP 和插件自动化场景里，经常需要把截图、OCR 原图、生成结果或调试产物临时持久化，并拿到一个可被前端、IM 或 API 直接访问的 URL。对象存储 + CDN 稳定但接入成本高；公共图床 API 对自动化不友好，鉴权、限流、回收策略都不透明。GitHub 仓库 + jsDelivr 是低成本、可版本化、可脚本化的折中方案，适合小规模工具链。

## 问题

直接用 GitHub raw 链接会遇到几个问题：访问延迟不稳定、部分网络下不可达、raw 的 Content-Type 对图片预览偶尔不友好。jsDelivr 作为公共 CDN 可以读取 GitHub 仓库与 Release 文件，并返回更适合 Web 使用的响应头，同时提供多节点加速。但如果不控制版本、不限制文件大小、不处理缓存，很快就会踩坑。

## 做法

### 1. 建独立 asset 仓库

不要混入主项目代码。用单独仓库专门放图片或二进制产物，避免污染主仓库历史和 CI。仓库必须 public，否则 jsDelivr 无法读取。

### 2. 上传图片

小批量可以直接用 Git 提交：

```bash
git add assets/img/ && git commit -m "add img" && git push
```

自动化场景建议用 GitHub API：

```text
PUT /repos/{owner}/{repo}/contents/{path}
```

把图片 base64 后写入，或通过 Release 上传接口托管较大文件。Agent/MCP 可以把“截图 → base64 → 提交到 asset repo → 返回 CDN URL”封装成一个工具。

### 3. 生成 jsDelivr URL

格式：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{tag|commit|branch}/{path}
```

例如：

```text
https://cdn.jsdelivr.net/gh/yourname/assets@main/img/2025/foo.png
```

建议自动化输出使用 commit hash 或 tag，而不是 branch。这样下次更新不会让旧 URL 内容突变。

### 4. 代理与回退

如果主 CDN 域名不稳定，可替换为：

```text
https://fastly.jsdelivr.net/gh/{owner}/{repo}@{version}/{path}
```

或暂时回退到 raw.githubusercontent.com。把 CDN 域名做成配置项，而不是硬编码。

## 踩坑点

- **仓库/blob 膨胀**：频繁提交图片会让 .git 历史快速增长。使用独立仓库、定期压缩，或大文件走 Release 资产。
- **缓存不即时**：jsDelivr 对 GitHub 分支路径有缓存，更新后旧图可能仍命中。需要强制刷新时可调用 purge 接口，但自动化更推荐不让同一路径被覆盖。
- **文件大小限制**：建议图片压缩为 WebP/PNG，单张控制在几 MB 以内；不适合放视频或超大素材。jsDelivr 适合轻量预览，不适合原图归档。
- **隐私与合规**：仓库必须公开，不能放敏感截图、内部数据或未脱敏 OCR 内容。
- **稳定性预期**：免费公共 CDN 没有 SLA。生产级关键链路仍建议 OSS/CDN，此方案更适用于内部工具、开发联调、Bot 输出预览和少量共享。

## 可复用建议

- 按内容哈希命名：`sha256(img)[:12].webp`，避免缓存问题和重复上传。
- 写一个 MCP 工具或 Agent action：输入本地图片路径或 base64，输出 `cdn.jsdelivr.net` URL 和 fallback URL。
- 用 GitHub Actions 做图片瘦身：提交前统一转 WebP、压缩，避免手工上传原始大图。
- 仓库只放 public 资产，README 中声明文件可被公共 CDN 访问，避免后续误解。

## 总结

GitHub + jsDelivr 免费图床的价值不在性能，而在于“仓库即图床”的可编程性：图片可版本化、可通过 API 写入、可被 Agent 工具链直接调用。控制好公开性、文件大小和缓存策略后，它在 OpenClaw/Agent/MCP 等自动化实践里是一个轻量、可复用的图片托管层。不要把它当生产 CDN，但作为工程脚本的默认图片输出通道足够实用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b1b84caf98cb5139.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/632974a8bb4089a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b4307ca8a7210d4c.png)

