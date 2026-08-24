---
title: GitHub + jsDelivr 搭免费图床：给 OpenClaw/Agent 工作流一个可版本化的图片直链方案
feedId: 34559
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP、插件这类自动化项目里，图片外链出现的频率比想象中高：插件 README 预览图、MCP 返回的卡片封面、自动化报告截图、机器人消息配图、配置项里的图标 URL。如果这些图片挂在临时图床、聊天工具相册或第三方平台，很容易遇到直链失效、防盗链、图片被压缩、无法版本管理等问题。

把图片当作代码资产的一部分，用 GitHub 仓库托管，再通过 jsDelivr 提供 CDN 访问，是一种链路透明、成本低、可脚本化的方案。它不适合替代生产级对象存储，但对个人项目、小团队内部工具和自动化流程来说，已经足够实用。

## 问题

我在维护 OpenClaw 相关小工具时，主要遇到几个痛点：

- 图片直链不稳定，需要登录或带临时 token 才能访问。
- 图片更新后，CDN 缓存刷新不可控，旧 URL 可能继续显示旧图。
- 图片无法和项目版本绑定，回滚代码时图片对不上。
- 自动化流程里缺少稳定的上传和取链方式，手动上传很麻烦。
- 某些海外图床在国内访问慢，影响 README 和 MCP 输出展示。

## 做法与步骤

### 1. 建一个公开仓库作为图床

GitHub 仓库必须是 `public`，否则 jsDelivr 无法拉取。可以单独建一个 `assets` 仓库，也可以放在项目仓库的 `assets` 目录下。单独仓库的好处是不会污染主项目提交历史。

建议目录结构：

```text
assets/
  agents/
    demo/
      cover.webp
      icon.png
  reports/
    2025-01/
      summary.webp
```

### 2. 上传图片

手动上传可以直接拖到 GitHub 网页，但更推荐脚本化上传。可以用 `gh` 命令通过 GitHub Contents API 传 base64：

```bash
REPO=yourname/assets
BRANCH=main
FILE=assets/agents/demo/cover.webp

gh api repos/$REPO/contents/$FILE \
  -f message="add cover" \
  -f content="$(base64 -w0 cover.webp)" \
  -f branch=$BRANCH
```

也可以在 GitHub Actions 里配置 `contents: write` 权限，在 release 或 push 后自动压缩、上传图片。

### 3. 构造 jsDelivr URL

URL 格式为：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>
```

例如：

```text
https://cdn.jsdelivr.net/gh/yourname/assets@main/assets/agents/demo/cover.webp
```

生产环境建议不要用 `@main` 或 `@latest`，而是固定 tag 或 commit hash：

```text
https://cdn.jsdelivr.net/gh/yourname/assets@v0.1.0/assets/agents/demo/cover.webp
https://cdn.jsdelivr.net/gh/yourname/assets@3f2a1b9/assets/agents/demo/cover.webp
```

固定版本后，更新图片时先生成新 URL，避免旧缓存影响。

### 4. 接入自动化流程

在 OpenClaw/Agent/MCP 链路中，可以把上传逻辑封装成一个脚本或 MCP 工具：接收截图或生成图，压缩后推到 assets 仓库，返回 jsDelivr URL 和 GitHub raw URL 作为备份。这样 Agent 产出的图片可以自动落到可版本化的图床里。

## 踩坑点

1. **仓库必须公开**  
   私有仓库 jsDelivr 无法访问，这是最容易忽略的点。

2. **单文件大小限制**  
   GitHub 单文件不能超过 100MB，jsDelivr 对超大文件也不友好。图片应压缩到几百 KB，WebP 优先，透明图标用 PNG。

3. **缓存更新不是即时生效**  
   `@main`、`@latest` 这类动态版本可能存在缓存延迟，同名覆盖后旧 URL 可能长时间返回旧图。固定 commit hash 或 tag 是最稳的做法。

4. **国内可用性有波动**  
   jsDelivr 在国内有节点，但偶尔会出现 DNS 污染或访问慢。关键展示链路建议增加 raw GitHub URL 或其他 CDN 作为冗余。

5. **文件名大小写敏感**  
   `Cover.webp` 和 `cover.webp` 是两个路径，URL 必须完全一致。

6. **删除或重命名会永久失效**  
   它不会像对象存储那样做 301 跳转，旧链接会直接 404。因此不要轻易删除或改名已发布的图片。

7. **不要承载大流量生产业务**  
   jsDelivr 有合理使用政策，滥用可能被限制。生产环境还是应该用对象存储 + CDN。

## 可复用建议

- 图片先压缩，单张控制在 500KB 以下，优先 WebP。
- 统一命名：小写、连字符、语义化，避免时间戳乱码。
- 按模块和日期分目录，避免单个目录堆积。
- URL 固定到 commit hash 或 tag；README 里写明替换规则。
- 上传脚本返回 jsDelivr URL 时，同时打印 GitHub raw URL 作为回退。
- 对关键图片做简单健康检查，例如定期 `curl -I` 检查状态码。
- 如果项目需要高可用，对象存储 + CDN 仍然是首选，这个方案适合个人、小团队和内部工具。

## 总结

GitHub + jsDelivr 作为免费图床，优势在于免费、可版本化、可脚本化，能和代码资产同源管理。缺点是缓存更新有延迟、国内访问偶尔波动、删除后链接不可恢复。对于 OpenClaw/Agent/MCP 这类自动化工作流中的临时展示、内部报告、插件预览等场景，它是一个务实且可控的选择。生产关键链路仍然需要引入对象存储和多 CDN 冗余。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ffb570fe202f8176.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/5ca81d1869fa5c43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/78894a69c0aa3bba.png)

