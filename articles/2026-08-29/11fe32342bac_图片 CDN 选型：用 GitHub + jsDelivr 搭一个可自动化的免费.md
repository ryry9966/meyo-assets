---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个可自动化的免费图床
feedId: 35229
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent、MCP 和插件自动化实践里，经常要把运行结果转成图片：任务截图、生成式图片、预览图、日志快照、异常现场留档。这些静态资源需要一个稳定、可编程访问的 URL，方便被 Markdown、卡片、通知或下游工作流直接引用。

自建 OSS 需要维护权限、桶策略和账单；第三方图床不透明、接口随时可能变化；把图片塞进消息队列或数据库又会污染主流程。GitHub 公共仓库 + jsDelivr 的组合提供了一种折中：用 Git 管理图片，用公共 CDN 输出，天然适合和自动化流水线集成。

## 问题

直接用 GitHub 的 raw 地址有两个明显短板：一是 `raw.githubusercontent.com` 在部分网络环境下访问慢；二是它没有 CDN 缓存语义，频繁请求不合适。jsDelivr 可以把任意公共 GitHub 仓库文件转换为：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch|commit|tag}/{path}
```

它免费、全球分发、无需额外注册，对低频、小规模、可审计的图床需求足够务实。

## 做法/步骤

### 1. 建一个公共仓库

在 GitHub 创建 public 仓库，例如 `assets` 或 `openclaw-images`。图片按日期或任务 ID 分目录存放：

```text
2025/01/task-1234.png
```

避免根目录堆积，也方便后续清理和审计。

### 2. 生成最小权限令牌

进入 GitHub Settings → Developer settings → Personal access tokens，创建一个只勾选 `repo` 下 `Contents` 读写权限的 token。把 token 放到运行环境变量 `GITHUB_TOKEN`，不要写进脚本、日志或提交到仓库。

### 3. 上传图片并生成 CDN URL

手动上传可以直接用 `gh` 或 git：

```bash
gh repo clone yourname/assets /tmp/assets
cp result.png /tmp/assets/2025/01/task-1234.png
cd /tmp/assets
git add . && git commit -m "add task image" && git push
```

拿到 commit SHA 后，图片地址就是：

```text
https://cdn.jsdelivr.net/gh/yourname/assets@<commit-sha>/2025/01/task-1234.png
```

在 Agent/MCP 工具里，可以封装一个 `upload_image` 函数：复制图片到本地 checkout，生成唯一文件名，提交并推送，最后返回 jsDelivr URL。相比直接调用 GitHub Contents API，用 `git` 或 `gh` 更直观，也不容易在 base64 编码上出错。

### 4. 接入 OpenClaw 工作流

例如定义一个 `save_image_to_cdn(path)` 工具，从环境变量读取 `GITHUB_TOKEN`，执行上传并返回 `@commit-sha` 的 CDN URL。这样图片生成、上传、引用就变成一条可复用的自动化链路，而不是每次手动传图。

## 踩坑点

- **缓存刷新**：使用 `@main` 或 `@latest` 这类可变分支时，jsDelivr 会缓存内容。更新同名文件后，旧图可能残留。尽量使用 commit SHA 作为版本，保证 URL 内容一致；如果必须用分支，可以在 URL 后加 `?v=时间戳` 破坏缓存，但不推荐。
- **文件大小限制**：jsDelivr 单文件限制约为 20MB，GitHub 仓库单文件限制 100MB。它适合截图、预览图和普通图片，不适合大视频或高分辨率原图。
- **只支持公共仓库**：私有仓库无法通过 jsDelivr 访问，选择时要确认资源可以公开。
- **文件名编码**：路径含空格、中文或特殊字符必须做 URL 编码，否则会 404。建议统一使用小写、连字符命名，例如 `task-1234-preview.png`。
- **API 上传坑**：如果改用 GitHub Contents API，base64 字符串中的换行符会导致图片损坏。需要去除换行，并正确设置 `Content-Type: application/vnd.github+json`。
- **网络稳定性**：jsDelivr 在某些区域可能间歇性不可用。生产依赖建议准备备用域名，如 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`，或准备其他静态托管作为兜底。不要把关键对客服务完全押在免费 CDN 上。

## 可复用建议

- 固定 `@commit-sha`，不要使用 `@main` 作为长期引用。
- token 使用最小权限，并设置过期时间；放在环境变量、密钥管理或 CI secret 中。
- 上传前用 `pngquant`、`imagemagick` 或插件内置压缩能力控制体积，单张图尽量在几百 KB 以内。
- 在 Agent 工作流中把图片 URL 作为返回值传递，而不是在不同节点间搬运图片二进制。
- 写一个简单清理脚本，定期删除 30 天未引用图片，避免仓库膨胀。

## 总结

GitHub + jsDelivr 并不是一个“高可用生产图床”，但它非常适合 Agent/MCP/自动化场景中的静态资源出口。它免费、可编程、可审计，能够和 Git 流程自然融合。只要接受它的容量和稳定性边界，用 commit hash 固定内容、控制图片尺寸、保留备用方案，图片托管就能从运维负担变成几行脚本的小事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/276727c453a1e2ee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e9d72a6303e61273.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fce5ca5c9aad0d77.png)

