---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭建可版本化免费图床的实践
feedId: 34566
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP server、插件自动化这一类场景里，经常需要把本地生成的截图、渲染卡片、状态徽章或流程图发布成 HTTP URL，供 Markdown 报告、通知消息或前端页面消费。之前试过临时图床，问题比较明显：外链容易失效、看不到历史版本、上传动作不可复用、无法接入自动化流水线。

后来我把图片分发固定到 GitHub 仓库 + jsDelivr CDN 这条链路上。核心诉求并不是追求毫秒级可用性，而是：**可版本管理、可脚本化上传、小体积图片、能接受公开访问**。它适合作为轻量免费图床，但不适合当作强依赖生产 CDN。

## 问题

这个选型需要解决几个具体问题：

- 图片在自动化流程里生成后，要能一键发布，不能每次手动传网页。
- URL 要稳定，要能区分版本，避免同名覆盖后出现缓存错乱。
- 仓库配额和 CDN 缓存策略要可控。
- 国内部分网络下可用性不确定，因此只放可公开、非强依赖资源。

## 做法 / 步骤

1. **建 public 仓库**

   比如建一个 `image-hosting` 或 `assets` 仓库，目录按 `YYYY/MM` 或 `project/module` 组织。仓库必须 public，jsDelivr 才能通过 GitHub 公开仓库拉取文件。

2. **使用 jsDelivr 地址**

   提交图片后，访问格式为：

   ```text
   https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>
   ```

   例如：

   ```text
   https://cdn.jsdelivr.net/gh/yourname/image-hosting@main/2025/04/report-20250412-a1b2c3d4.webp
   ```

3. **自动化上传**

   在 OpenClaw/Agent 侧封装一个脚本或 MCP 工具，输入本地图片路径，自动完成压缩、重命名、复制到 git 仓库、commit、push，最后返回 jsDelivr URL。这样报告生成、截图回传、插件产物发布都能复用同一条链路。

4. **唯一命名**

   文件名使用类似 `<project>-<yyyymmdd>-<sha1前8位>.webp` 的规则，避免同名覆盖。唯一命名比依赖 CDN 缓存刷新更可靠。

5. **版本化引用**

   对需要长期保留的图片，建议使用 tag 或 commit sha 引用：

   ```text
   https://cdn.jsdelivr.net/gh/yourname/image-hosting@v1.0.0/path/to/img.webp
   ```

   避免长期使用 `@main` 作为不可变资源引用，否则同名更新后历史报告可能指向不同内容。

6. **缓存刷新**

   如果确实需要更新同名文件，可以调用 jsDelivr 的 purge 接口：

   ```text
   https://purge.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>
   ```

   但实际使用中，我更推荐“新文件名”策略，减少对 purge 的依赖。

## 踩坑点

- **仓库体积**：GitHub 仓库超过 100MB 会收到警告，非常大会影响拉取体验。图床图片应尽量压缩，单张建议控制在 300KB 以内。
- **缓存时间**：jsDelivr 对 GitHub 资源的缓存时间较长，同名文件更新可能不会立刻生效。即使 purge，边缘节点刷新也需要时间。
- **公开边界**：public 仓库的所有文件都能被拉取，绝不能存含 token、内部地址、用户隐私的截图。
- **文件名规范**：只用小写字母、数字、连字符、下划线，避免空格、中文、`#`、`?` 等需要编码的字符。
- **GitHub API 限流**：未认证 API 限流约 60 次/小时。自动化频繁上传时建议带 PAT，并控制提交频率。
- **国内可达性**：jsDelivr 在国内部分网络环境可能不稳定，不要作为核心业务强依赖。可备选 `fastly.jsdelivr.net`，或本地 OSS/Nginx 兜底。
- **Actions 递归触发**：如果用 GitHub Actions 自动提交，注意 `on: push` 可能触发自身再次运行，需要设置路径条件或使用 `workflow_dispatch`。

## 可复用建议

- 把“压缩 + 重命名 + commit + 返回 URL”封装成一个固定命令，例如 `image push --file ./render.png --project report`。OpenClaw 插件或 MCP server 只需调用这个入口。
- 图片统一转 WebP，宽度限制在 1200px 左右，CDN 加载和仓库体积都更可控。
- 图床上只放分发副本，原始文件保存在构建产物或独立备份中，不要让它成为唯一备份。
- 报告、通知、卡片中尽量使用 commit sha 版本的 URL，保证回溯时图片和代码版本一致。
- 定期清理历史大文件，必要时做 `git gc`，避免 `.git` 目录膨胀。

## 总结

GitHub + jsDelivr 适合低频、小体积、可公开、需要版本化的图片分发，尤其适合 OpenClaw/Agent/MCP 工作流里的报告卡片、运行截图、状态图等场景。它不是高可用生产 CDN，也不是私有图床。

关键工程点是：**唯一命名、压缩、版本化引用、自动化封装、明确公开内容边界**。做到这几点，这个免费方案可以稳定运行很长时间；如果不加控制，仓库膨胀和缓存错乱会比预期来得更快。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b1764e28311b8530.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c9a8a452016e5701.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/731dd85aa91ff153.png)

