---
title: 图片 CDN 选型：用 GitHub + jsDelivr 给 Agent 图片输出加一层免费 CDN
feedId: 34815
source: 综合讨论
publishedAt: 2026-08-26
---

# 图片 CDN 选型：用 GitHub + jsDelivr 给 Agent 图片输出加一层免费 CDN

## 背景

在 OpenClaw、Agent、MCP 和插件开发中，经常需要把生成图片变成可跨平台引用的 URL：截图、图表、二维码、预览图、中间产物等。临时目录和本地路径无法在上下游传递；自建对象存储又增加组件。对于小批量、低频、可公开的图片，GitHub 仓库 + jsDelivr 是一个低依赖、可版本化、可自动化的免费方案。

## 问题

直接用 `raw.githubusercontent.com` 访问图片，延迟不稳定，缓存控制也不适合频繁引用。jsDelivr 作为第三方 CDN，可以把 GitHub 仓库文件分发到边缘节点，并支持版本号、缓存刷新。但前提是仓库必须公开，文件大小、流量和缓存策略都有限制。选型前需要确认场景：适合 Agent 产出的预览图、示意图、二维码、截图等小文件，不适合视频、大图集或高频大流量。

## 做法/步骤

### 1. 准备公开仓库

创建一个公开仓库，例如 `assets` 或 `agent-images`。目录建议按年月或模块划分，避免根目录堆积：

```bash
mkdir -p assets/2025/06
git add assets/2025/06/example.png
git commit -m "chore: add example image"
git push origin main
```

### 2. 使用 jsDelivr URL 规则

文件路径固定为：

```text
https://cdn.jsdelivr.net/gh/<owner>/<repo>@<tag>/<path>
```

建议用 tag 或 commit hash，不要用 `@latest`。例如：

```text
https://cdn.jsdelivr.net/gh/yourname/assets@v0.1.0/assets/2025/06/example.png
```

这样同一 URL 不会因主干更新而变化，适合下游缓存。

### 3. 在 Agent/MCP/插件中自动上传并返回 URL

最直接的方式是在生成图片后调用 GitHub MCP 工具，例如 `create_or_update_file`，把二进制转 base64 后写入仓库。然后按模板拼接 URL 返回给调用方，而不是只返回本地路径。

如果使用 GitHub API，可以用类似命令：

```bash
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"upload image\",\"content\":\"$(base64 -w0 image.png)\"}" \
  "https://api.github.com/repos/OWNER/REPO/contents/assets/2025/06/image.png"
```

在 TypeScript/Node 插件中，建议封装一个纯函数：

```ts
function jsDelivrUrl(repo: string, tag: string, filePath: string) {
  return `https://cdn.jsdelivr.net/gh/${repo}@${tag}/${filePath}`;
}
```

这样 Agent 每次只关心“传哪张图、返回哪个 URL”，其余细节被固化在工具里。

## 踩坑点

- **仓库必须公开**：jsDelivr 不支持私有仓库，敏感图片不要传。
- **缓存刷新**：修改同一路径的图片后，jsDelivr 可能继续返回旧内容。可以用 `https://purge.jsdelivr.net/gh/<owner>/<repo>@<tag>/<path>` 强制刷新，或用内容哈希命名新文件，彻底避免覆盖。
- **文件大小**：单文件建议控制在 10–20MB 以内；更大文件会被拒绝或影响 CDN 表现。上传前用 `sharp` 或 `imagemin` 压缩成 WebP/PNG。
- **路径合规**：文件名大小写敏感，避免空格和特殊字符，URL 需要 encode。
- **国内可用性**：jsDelivr 在大陆偶尔会波动。可以在自动化里做 fallback，例如同时返回 `raw.githubusercontent.com` 链接，但不要把 jsDelivr 当作唯一关键路径。
- **仓库膨胀**：不要频繁覆盖或删除大二进制文件，git 历史会变大。可定期重建仓库或用 Git LFS（注意 jsDelivr 对 LFS 支持有限，不推荐）。

## 可复用建议

1. **固定目录规范**：`assets/<project>/<yyyy-MM>/<content-hash>.<ext>`，避免重名和缓存混乱。
2. **自动化压缩**：在 GitHub Actions 里对 push 的图片跑压缩和格式转换，再提交优化版本。
3. **URL 模板写进插件配置**：不要把拼接逻辑散落在 Agent prompt 里，集中到一个 MCP 工具或配置项。
4. **显式声明公开性**：在仓库 README 写明“本仓库图片会通过公开 CDN 分发”，降低误传隐私风险。
5. **监控容量**：GitHub 仓库建议不超过 1GB，可设置 Actions 或脚本定期检查大小。

## 总结

GitHub + jsDelivr 不是高性能图床，但在 OpenClaw/Agent/MCP/插件的小批量图片分发场景里，足够轻量、可复现、易自动化。关键是把“上传”和“返回 URL”封装成工具，并遵守公开仓库、版本化、内容哈希和大小限制这些边界。只要不越界，它可以成为自动化链路里一个很顺手的免费 CDN 组件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/57b14fcf276c92b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b311ef17f18136fb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b206fec7f306c4af.png)

