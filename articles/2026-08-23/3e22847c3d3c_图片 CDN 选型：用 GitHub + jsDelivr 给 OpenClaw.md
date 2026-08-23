---
title: 图片 CDN 选型：用 GitHub + jsDelivr 给 OpenClaw 产物搭免费图床
feedId: 34383
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw/Agent/MCP 的自动化流程里，经常会出现图片型中间产物：架构图、截图、OCR 结果、插件生成的封面、数据图表等。它们如果只停留在本地路径，后续写入 Markdown、推送到聊天平台或作为 API 返回内容时就会失效；如果直接塞进主项目仓库，仓库体积又会被快速撑大。

因此需要一个能通过 API 上传、返回稳定的 URL、并且尽量少维护的图床。免费公共图床大多需要网页登录、接口不透明，不适合 Agent 自动调用。相比之下，GitHub 仓库 + jsDelivr 的组合更适合小规模、可版本管理的自动化图床。

## 问题与选型

核心诉求是：

- Agent/MCP 能程序化上传，而不是手点网页；
- 返回的 URL 可预测、可拼接；
- 免费额度足够日常小图；
- 有一定 CDN 加速能力。

GitHub raw 在国内访问不稳定，而 jsDelivr 可以把 GitHub 仓库文件转换成 CDN URL，规则固定。代价也很明显：仓库和单文件大小受限，同名覆盖时存在缓存延迟。

如果图片总量控制在几百 MB 以内、单图压缩到 1–2MB，这个方案足够务实；如果是量产大图或高频覆盖更新，建议后续迁移到对象存储，但工具接口可以保持不变。

## 做法/步骤

**1. 建仓和 token**

新建一个 public 仓库，例如 `image-host`，不要和业务主仓库混用。目录按日期拆分，避免单目录堆积过多文件：

```text
assets/2025/02/14/a1b2c3.png
```

Token 使用 fine-grained PAT，只授予该仓库的 Contents 读写权限。不要给 repo 之外权限，也不要长期使用经典 PAT。

**2. 上传 API**

上传直接调 GitHub Contents API：

```text
PUT /repos/{owner}/{repo}/contents/{path}
```

请求体包含 `message`、`branch`、`content`，其中 `content` 是图片的 base64 编码。成功后返回 `content.sha`，可以把短 commit hash 记录下来。

在 OpenClaw 里，建议把上传封装成 MCP 工具，例如 `upload_image(path)`，内部读取 `GITHUB_IMAGE_TOKEN` 环境变量，压缩后上传，并返回 CDN URL。这样 Agent 生成图片后直接调用工具即可，不需要把图片内容塞进上下文。

如果不想写 MCP，也可以用 GitHub Actions 在 push 后自动压缩并 commit。但对 Agent 实时上传场景，MCP 工具延迟更低。

**3. URL 拼接**

jsDelivr 的规则为：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

上传后首次访问可能较慢，因为 CDN 需要回源。可以在上传完成后用 `curl -I` 验证状态码是否 200。

**4. 版本化引用**

日常调试可以用 `@main`，但长期引用或发布到文档、卡片时，建议把 `@main` 换成 commit hash：

```text
https://cdn.jsdelivr.net/gh/user/image-host@{commit}/{path}
```

这样即使以后覆盖同名文件，旧 URL 仍然指向旧版本，不会出现图片突然变化。

## 踩坑点

- **同名覆盖缓存**：`@main` 的 CDN 缓存最长可能到 12 小时。短时间内覆盖同名文件，用户看到的还是旧图。要么文件名带内容 hash，要么使用新 commit hash 的 URL。也可以用 `purge.jsdelivr.net`，但频率受限，不适合作为常规方案。
- **文件大小限制**：jsDelivr 对 GitHub 文件约 50MB，GitHub 仓库本身建议不超过 1GB。上传前先用 sharp/imagemagick 压缩，限制最大宽度并转 WebP。
- **Token 泄露**：不要把 token 写进 prompt、日志或前端。使用环境变量或 secret 管理，并限制到单个仓库。
- **仓库不可过大**：大量图片写入同一仓库可能导致 CDN 变慢或触发 GitHub 策略。建议按月归档，必要时拆分仓库。
- **国内可访问性**：jsDelivr 大部分时间可用，但不是 SLA 保障。关键图片应有对象存储或自建服务兜底。

## 可复用建议

- 固定命名规范：`{date}/{hash}.{ext}`，避免重名和路径注入。
- 上传前压缩：在 MCP 工具内部做一道 `sharp` 压缩，限制最大宽度 1600px，输出 WebP。
- 上传后验证：返回 URL 前用 `HEAD` 请求确认 CDN 可访问；失败则 fallback 到 GitHub raw URL 并告警。
- 长期引用使用 commit hash URL，临时调试用 `@main`。
- 记录每个图片的 path、sha、创建时间，定期清理未引用图片。

## 总结

GitHub + jsDelivr 不是高性能图床，但它在免费、API 可操作、版本可管理之间取了一个很实用的平衡。对 OpenClaw/Agent 自动化来说，把图片上传封装成 MCP 工具，内部使用受限 token、压缩规则和版本化路径，能把“图片从哪来、放在哪、URL 可不可用”这些杂音大幅降低。后续图片量增长时，再切换对象存储即可，工具接口和 URL 规则不需要推翻重来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/83ff9b21a1fdb437.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a411c4750eb3341f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/97c6c54d511b401d.png)

