---
title: GitHub + jsDelivr 免费图床的自动化实践：从上传到 CDN 的工程方案
feedId: 31278
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：为什么 OpenClaw 场景需要可控的图床

在 OpenClaw 的 Agent、MCP 与自动化流程中，图片正逐渐成为交互载体——Agent 生成的结果里嵌示意图，自动化报告需要截图附件，MCP 插件返回的上下文也会包含图片链接。这些场景对图片托管有几个硬性要求：

- **链接稳定**：不能今天能用明天失效，导致 Agent 输出或自动化流程断层。
- **可控上传**：最好是 API 或命令行即可完成，不依赖人工在 Web 端拖拽。
- **成本趋近于零**：多数个人或小团队项目无法为随机的图片存储支付额外账单。

常见的免费图床要么限制外链（如 imgur 的匿名上传），要么在境内访问不稳定，要么无法通过自动化上传。自建 MinIO 或阿里云 OSS 虽然可靠，但需要维护服务器和域名，对轻量级项目来说太重。

因此，我们将目光投向开发者最熟悉的组合：**GitHub 仓库 + jsDelivr CDN**。这并非新点子，但围绕 OpenClaw 的自动化实践，需要补全从脚本上传、链接构造到踩坑复盘的完整链条。

## 问题：需要一套“上传门槛低、CDN 加速”的管道

我们的主要诉求是：

1. 把图片以自动化方式存入一个稳定存储（无需自建服务）。
2. 拿到一个能直接 HTTP 访问、全球加速的 URL。
3. 尽量不引入复杂的认证逻辑，避免 Token 泄露风险。

GitHub 仓库天然支持 Raw 文件访问，但直连速度慢且偶有阻断。jsDelivr 能镜像 GitHub 仓库内容并提供多地 CDN 加速，免费、免审核、直接走 `cdn.jsdelivr.net` 域名。于是一个典型的流水线便成立了：**本地或 Agent 端产生图片 → 推送至 GitHub 仓库 → 用 jsDelivr 的路径规则构造 CDN 链接**。

## 做法与步骤

### 1. 创建专用图床仓库并初始化

在 GitHub 创建一个 Public 仓库（Private 仓库的文件无法通过 jsDelivr 访问，因为 jsDelivr 只能镜像公开仓库）。建议仓库名如 `assets`，并提交一个 `.gitkeep`，方便后续直接拉取。

在本地克隆该仓库，目录结构按需划分，例如：

```
images/
  2024/
    screenshot1.png
```

> 注意：jsDelivr 对 GitHub 文件大小限制为 20MB，但免费仓库总容量 1GB，足够大多数 Agent 图像产出。

### 2. 自动化上传脚本

以 Python 脚本为例，通过 PyGithub 或直接调用 GitHub API 上传图片。如果不想在本地管理 Token，可以在 CI 环境（如 GitHub Actions）里使用 Repository Secret，或者依赖系统已经配置好的 Git 凭证。

一个最简自动化思路是用 `git` 命令直接提交，适用于 Agent 所在环境已经拥有 Git 操作权限：

```bash
#!/bin/bash
REPO_PATH="/path/to/assets-repo"
IMAGE_PATH="$1"
FILENAME=$(basename "$IMAGE_PATH")
cp "$IMAGE_PATH" "$REPO_PATH/images/$FILENAME"
cd "$REPO_PATH"
git add .
git commit -m "add image $FILENAME"
git push origin main
```

再用 jsDelivr 规则生成链接。对于文件路径 `images/2024/screenshot1.png`，CDN URL 格式为：

```
https://cdn.jsdelivr.net/gh/用户名/仓库名@main/images/2024/screenshot1.png
```

`@main` 表示分支，可以指定具体 commit 或 tag 以控制缓存刷新。

### 3. 在 MCP 或 Agent 工具中集成

针对 OpenClaw 用户，一个更工程化的做法是将上传逻辑封装成一个 MCP server 的 tool。示例：工具 `upload_image_to_cdn` 接收图片的本地路径或 base64，然后：

- 用 Octokit (JavaScript) 或 PyGithub 直接创建仓库文件（PUT `/repos/{owner}/{repo}/contents/{path}`），并指定 commit message。
- 返回构造好的 jsDelivr URL。

这样可以避免在 Agent 所在环境中维护完整的 Git 仓库和历史，减少磁盘占用。注意 GitHub API 速率限制（鉴权请求 5000 次/小时），对图片频繁写入的场景可能需要合并推送或使用 Actions 批量处理。

## 踩坑点

**1. 缓存刷新不及时**

jsDelivr 的缓存周期较长, 同一个 URL 更新文件后不会立即生效。官方推荐的最佳实践是使用版本化的 tag 代替分支，或在 URL 中加入 `@版本号`。我们的做法是：每次上传后生成一个唯一的 commit hash 作为版本，但这样会让图片链接过长。折衷方案：对于不频繁更新的稳定资源，使用 `@latest`（即默认分支）是可接受的，但必须容忍最长 24 小时的缓存延迟。更适合自动化场景的是用 `@commitID` 精确引用，Agent 也更容易管理版本。

**2. 仓库文件大小限制**

GitHub 单文件限制 100MB，但 jsDelivr 限制为 20MB。若 Agent 生成的截图超过该大小，需要提前压缩。我们统一在 MCP 工具中加入 sharp(pngquant) 压缩步骤，保证文件 < 5MB。

**3. 路径大小写与特殊字符**

因为 GitHub 仓库文件系统大小写敏感，而 jsDelivr 会遵循源路径。如果 Windows 环境默认忽略大小写，可能导致 URL 与文件实际路径不符。强制所有文件名小写并用连字符，回避特殊字符。

**4. Public 仓库暴露风险**

即使用 Public 仓库，所有图片都是公开可访问的。不适合存放任何敏感截图或隐私数据。我们建议仅存放 Agent 产出的非敏感图表，如数据可视化、示例图解等。敏感内容应该走其他私有图床或临时链接方案。

## 可复用建议

- **轻量级图片 CDN 即服务**：可以将该方案打包成一个独立 MCP server（GitHub 作为后端），为多个 Agent 实例提供图片托管，返回 `https://cdn.jsdelivr.net/gh/...` 前缀的稳定链接。
- **搭配 GitHub Actions 做后处理**：如果图片需要转换格式或水印，在仓库上配置 Actions，监听 push 事件后自动处理，再更新文件，最终用处理后的 CDN 链接。
- **监控容量与清理策略**：设置 Actions 定期检查仓库大小，当接近 1GB 时发出提醒，并提供自动清理老旧图片的选项。
- **避免单点依赖**：虽然 GitHub + jsDelivr 比较稳定，但仍建议半年左右备份一次整个 assets 仓库到其他地方（如 Cloudflare R2），以防服务变更。

## 总结

GitHub + jsDelivr 组合提供了一种极低成本的图床方案，特别契合 OpenClaw 自动化环境下对稳定、API 可操控图片存储的需求。它的优势在于零额外费用、能与现有 Git 工作流或 MCP 工具无缝集成；劣势是公开访问和缓存延迟。通过合理的流程设计（压缩、版本化 commit、定期备份），足以承载大量非敏感 Agent 图像产出的托管任务。对于需要在自动化管道中嵌入图片链接的开发者，这套方案值得作为默认轻量级选项。

---

