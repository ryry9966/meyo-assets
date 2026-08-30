---
title: 免费图床的工程化实践：用 GitHub 仓库 + jsDelivr 做图片 CDN
feedId: 35409
source: 综合讨论
publishedAt: 2026-08-30
---

# 免费图床的工程化实践：用 GitHub 仓库 + jsDelivr 做图片 CDN

## 背景

在 OpenClaw / Agent / MCP 的自动化流程里，经常需要把本地生成的图片（截图、图表、二维码、验证码、卡片预览等）快速转成可公开访问的 URL。手动上传到某家图床或对象存储，要么依赖不可控的第三方，要么需要维护密钥和计费。对于个人项目或小流量内部工具，GitHub 仓库 + jsDelivr 是一个零成本、可版本化的轻量方案。

## 问题

核心诉求很明确：  
1. 自动化脚本/插件负责上传图片，返回 URL。  
2. URL 可长期访问，最好带 CDN。  
3. 不引入复杂的云服务鉴权。  
4. 图片文件能溯源、能回滚。

但免费方案通常有代价：GitHub 仓库公开性、文件大小限制、CDN 缓存刷新延迟，以及部分网络环境下 jsDelivr 的可用性波动。踩过坑之后，这个方案更适合定位为“小流量、非关键业务、可接受降级”的图床。

## 做法

### 1. 创建专用公开仓库

新建一个 GitHub 仓库，例如 `image-hosting`，设为公开。不要混用代码仓库，避免权限和容量污染。

### 2. 上传图片

可以用 git 命令行：

```bash
git clone https://github.com/yourname/image-hosting.git
cd image-hosting
mkdir -p 2025/01
cp /tmp/screenshot.png 2025/01/
git add .
git commit -m "add screenshot"
git push origin main
```

如果接入了 MCP 或自动化插件，可以直接调 GitHub API 或使用 git 库。例如在 Python 中用 `PyGithub` 或直接 `git` 子进程。

### 3. 组装 jsDelivr URL

jsDelivr 对 GitHub 仓库的 CDN URL 格式为：

```text
https://cdn.jsdelivr.net/gh/用户名/仓库名@分支或版本/文件路径
```

例如：

```text
https://cdn.jsdelivr.net/gh/yourname/image-hosting@main/2025/01/screenshot.png
```

注意 `@main` 是分支名，也可以用 tag 或 commit hash。如果省略 `@main`，默认使用仓库默认分支。为了缓存可控，建议显式指定分支或 commit hash。

### 4. 自动化封装

写一个轻量上传函数，内部做三件事：  
- 图片压缩（可用 `imagemagick` 或 `sharp`）。  
- 重命名避免覆盖（时间戳 + 内容哈希）。  
- push 后返回 jsDelivr URL。

这样任何 Agent 或 MCP 工具只需调用一个函数，返回可直接嵌入 Markdown 的链接。

## 踩坑点

**缓存刷新延迟**  
jsDelivr 对 GitHub 文件的缓存可能长达数小时甚至更久。如果你覆盖同名文件，旧 URL 不会立即更新。解决方案：永远不要覆盖，使用哈希命名；或者 URL 中带 commit hash，每次 push 后更新 commit hash 引用。我推荐前者，简单可靠。

**文件大小限制**  
jsDelivr 对 GitHub 仓库单个文件限制为 20MB，GitHub 网页上传限制 25MB。对于截图和普通图片没问题，但大图或动图需要压缩或缩小尺寸。另外 GitHub 仓库总容量建议控制在 1GB 以下，避免被限制。

**国内访问波动**  
`cdn.jsdelivr.net` 在某些网络环境下可能超时或证书异常。可以准备备用域名，如 `fastly.jsdelivr.net` 或 `gcore.jsdelivr.net`，在自动化脚本里做 fallback：先请求主域名，失败后切换备用域名。

**隐私与合规**  
仓库公开，任何图片都能被他人通过 URL 访问。不要放内部截图、用户数据或敏感信息。如果必须私有，这条路走不通，需要另选方案。

**自动化鉴权**  
推送需要 GitHub token。不要把 token 硬编码，使用环境变量或 Secrets 管理。如果仓库只有自己贡献，可以用 Personal Access Token 限定 repo 权限。

## 可复用建议

- 独立仓库，按日期/内容哈希组织目录，禁止覆盖同名文件。
- 上传前压缩图片，控制单个文件在 1MB 以内更稳妥。
- 封装一个返回 CDN URL 的函数，内部处理 fallback 域名。
- 用 commit hash 或 tag 固定版本，便于回滚和缓存控制。
- 配置 GitHub Actions，在 push 时自动做图片优化（例如使用 `sharp` 压缩、生成 WebP 副本）。
- 监控可用性：定时 HEAD 请求图片 URL，记录状态码和延迟，发现异常切换备用 CDN 域名或降级为 GitHub raw 链接。

## 总结

GitHub + jsDelivr 作为免费图床，工程上可行，尤其适合 OpenClaw、Agent 和 MCP 插件的轻量自动化场景。它省去了对象存储的付费和鉴权复杂度，但代价是缓存延迟、公开性和网络波动。只要接受这些限制，并用哈希命名、备用域名和监控脚本兜底，它可以成为个人项目里很顺手的图片 CDN 组件。生产环境或关键业务，还是建议回归专业图床或对象存储 + CDN。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a62120c70e812850.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/eab1e2537d6958f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9a7d8b90dc9598dc.png)

