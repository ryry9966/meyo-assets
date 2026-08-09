---
title: GitHub + jsDelivr 搭建可编程图床：给 OpenClaw 插件开发者的静态资源方案
feedId: 32319
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在 OpenClaw 生态下，无论是开发 MCP 服务器、Agent 工具，还是为插件生成动态知识卡，都频繁需要托管少量静态图片——截图、图标、配置引导图或运行时生成的可视化结果。要求很简单：能外链、HTTPS、长期可访问、最好免费。但现实是，免费对象存储往往附带短时域名（如七牛测试域名 30 天回收），公共图床服务要么限频限容量，要么就悄悄把老链接重定向到广告页。更棘手的是，插件自动生成的图片需要可编程上传，大多数图床的 API 设计并不友好。

于是我们回到一个被很多前端工程师当 CDN 用的老组合：GitHub 仓库 + jsDelivr。这个方案天然适合静态资源分发，并且可以通过 GitHub API 或 Actions 做成自动化管线，匹配 OpenClaw 用户的工程化习惯。

## 问题分析

直接裸用 GitHub raw 链接有两个致命缺陷：一是 `raw.githubusercontent.com` 在中国大陆部分地区解析不稳定，二是不支持自定义缓存策略，稍微频繁请求就容易被限流。而 jsDelivr 在全球有数百个节点，会反向代理 GitHub/NPM 仓库，免费提供合理的缓存头和 HTTPS。结合 GitHub Actions，能够实现「事件触发 → 生成图片 → 自动提交到仓库 → 通过 jsDelivr 提供加速链接」的闭环。这个闭环对插件开发者来说极具吸引力：插件运行可以上传产出物，无需另外维护 OSS。

但整套方案并非无脑适用，有几个工程细节需要提前正视。

## 做法与步骤

### 1. 创建专用图床仓库

在 GitHub 新建一个 public 仓库，建议命名类似 `static-assets` 或 `og-image-cdn`。**必须 public**，否则 jsDelivr 无法拉取。目录结构简单即可，按类型分文件夹：

```
assets/
  screenshots/
  icons/
  card/
```

仓库设置中 **不需要** 开启 GitHub Pages，jsDelivr 直接通过内容分发。

### 2. 手工上传与初次验证

将一张测试图 push 到仓库 `assets/screenshots/test.png`。访问：

```
https://cdn.jsdelivr.net/gh/你的用户名/仓库名@latest/assets/screenshots/test.png
```

注意路径格式：`/gh/:user/:repo@:version/:file`。`@latest` 会指向默认分支最新提交，也可以固定为某个 commit hash 或 tag，这对版本管理很重要。如果图片正常显示，说明通道已通。

### 3. 自动化上传（核心步骤）

对 OpenClaw 用户而言，图床不能只是手动的。这里提供两种自动化模式。

**模式 A：通过 GitHub Actions 在插件/Agent 运行结束后上传**

在插件项目中，假设运行后会生成 `output/card.png`。可以在同一仓库的 `.github/workflows/upload-asset.yml` 中，用 `EndBug/add-and-commit` 或直接调用 `git` 命令提交到图床仓库。需要先为图床仓库配置一个 Personal Access Token（细粒度权限，仅给 Contents 写），存入当前仓库的 Secrets。步骤示例：

- checkout 当前项目（含生成的图片）
- checkout 图床仓库到临时目录
- 将新图片复制到对应文件夹，以时间戳或内容哈希命名避免覆盖
- push 回图床仓库

这样每次插件运行（由 schedule 或 issue 事件触发），生成的图片便自动发布到 CDN。

**模式 B：通过 GitHub API 在上层应用直接提交**

如果你的 Agent 跑在容器或函数计算里，可以直接用 GitHub Contents API 上传文件。接口：

```
PUT /repos/{owner}/{repo}/contents/{path}
```

body 包含 base64 编码后的图片内容、commit message、branch 等。一个调用即可完成上传，上传后 jsDelivr 链接几分钟内即可访问新文件（首次请求会回源，之后缓存）。

### 4. CDN 缓存与版本控制

jsDelivr 缓存策略是**永久缓存**（一年），但可通过 purge 缓存 API 加速刷新。如果使用 `@latest`，当推送新图片时需注意：jsDelivr 可能在一段时间内仍返回旧版本。更好的实践是使用 commit hash 作为版本标签：每次上传后，记录最新 commit sha，然后 URL 用 `@<commit>`。可在自动化流程中获取上传 commit 的 sha 并作为输出返回给插件。这样保证链接的强一致性。

## 踩坑点

1. **单文件 100 MB 限制**：GitHub 会对大文件警告甚至拒绝 push。图床只应放 PNG/JPEG/WebP 这些压缩后的小文件，禁止直接 push 原始 PSD、视频。
2. **jsDelivr 对超大仓库的拉取限制**：单次内容分发如果文件过多、仓库过大，可能导致拉取超时。保持仓库轻量，定期归档旧图片到 release 或另一个仓库。
3. **缓存刷新不及时**：默认缓存时间极长，紧急更新时需要手动调用 `https://purge.jsdelivr.net/gh/...` 强制刷新，或使用版本化 URL 避开缓存。
4. **仓库 public 导致的隐私问题**：图床内的所有内容对 anyone 公开，不能存放任何非公开截图。敏感内容请用私有对象存储。
5. **并发上传竞态**：自动化并行任务可能同时对同一分支 push，导致 conflict。建议上传流程串行化，或每次新建分支，合并后再打 tag。

## 可复用建议

结合 OpenClaw 社区常见场景，这套方案可封装成以下组件：

- **MCP 工具**：一个 `upload-to-cdn` 工具，接收本地文件路径，返回 jsDelivr 链接。可供 Agent 在对话中调用，上传生成的知识卡图片。
- **GitHub Action 模板**：预置仓库，提供 `cdn-upload-action`，只需配置 TOKEN 和文件路径，一键集成。这样插件作者不必重复造轮子。
- **通用图床 URL 生成函数**：在 Node/Python 图片生成逻辑中，输出文件后自动调用上传并返回 CDN 链接，极大降低接入成本。

流量与可靠性方面，jsDelivr 作为免费 CDN 有一定波动，但并不比绝大多数家用图床差。如果你的插件日均 PV 在万级别以下，完全够用。若遇到大流量场景，建议在前再加一层自己的域名 CNAME 或直接迁移到付费 CDN。

## 总结

GitHub + jsDelivr 方案在静态资源托管领域走过了大量验证，对于 OpenClaw 的插件和 Agent 这类低频、小尺寸的图片分发场景，是成本最低、自动化程度最高的选择之一。它的核心优势不是「能存图」，而是通过 Git 工作流和 API 让图床成为可编程的发布管线，这与 MCP 工具和自动化插件的理念深度契合。需要注意的点主要是缓存刷新策略与仓库公开性带来的约束，但只要在设计时做好版本化 URL 和内容安全边界，就能稳定运行。最后建议所有图片 CDN 实践都遵循一条原则：不要将 CDN 当永久存储，把源文件保留在本地或 Git 历史中，CDN 仅是分发层。

---

