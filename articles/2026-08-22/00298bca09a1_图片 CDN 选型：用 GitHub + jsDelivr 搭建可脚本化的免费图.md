---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭建可脚本化的免费图床
feedId: 34215
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw / Agent / MCP / 插件自动化流程里，经常需要把生成的截图、图表、二维码、模型输出图转成可外链的 URL。临时图床会过期，对象存储要维护凭证和账单，公共图床又不可版本化。对于个人项目、文档配图、低流量自动化产物，GitHub 仓库 + jsDelivr CDN 是一个低成本、可脚本化、能纳入 Git 工作流的方案。

## 问题

选型时我主要关注四点：零成本、可自动化、有版本控制、URL 可预期。GitHub 提供文件存储和版本历史，jsDelivr 提供 CDN 加速和固定格式的 URL 映射。组合起来适合中小型自动化场景，但不适合大流量、高隐私或高频覆盖需求。

## 做法/步骤

### 1. 建专用 public 仓库

不要混在主代码仓库里，单独建一个 `assets` 或 `images` 仓库，并设为 public。私有仓库 jsDelivr 无法访问。

目录建议按用途和时间拆分：

```text
blog/2025-01/abc123.png
qr/2025-01/def456.png
```

### 2. 文件名使用内容 hash

jsDelivr 对同名文件有长缓存，更新图片时不要覆盖已有文件名。文件名建议带内容 hash，例如 `sha256sum` 前 10 位，或短 UUID。只保留小写字母、数字、连字符，避免空格、中文和特殊字符。

### 3. 上传脚本

在 OpenClaw 插件或 MCP 工具中调用上传脚本，接收本地图片路径，返回 CDN URL。示例：

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO="your/assets"
BRANCH="main"
FILE="$1"
HASH=$(sha256sum "$FILE" | cut -c1-10)
EXT="${FILE##*.}"
NAME="$(date +%Y-%m)/${HASH}.${EXT}"
mkdir -p "$(dirname "$NAME")"
cp "$FILE" "$NAME"
git add "$NAME"
git commit -m "upload ${NAME}"
git push origin "$BRANCH"
URL="https://cdn.jsdelivr.net/gh/${REPO}@${BRANCH}/${NAME}"
echo "$URL"
```

### 4. URL 转换规则

GitHub raw 地址：

```text
https://raw.githubusercontent.com/<user>/<repo>/<branch>/<path>
```

对应 jsDelivr 地址：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

务必固定 `@<branch>`，不要省略，避免默认分支变更导致链接失效。

### 5. 可选：生成压缩版

如果图片较大，可以在上传前用 `sharp`、`pngquant` 或 `mozjpeg` 压缩到 1MB 以内。也可以在 GitHub Actions 中自动压缩，避免仓库膨胀。

## 踩坑点

- **缓存不刷新**：jsDelivr 没有 purge API，同名路径可能缓存很久。更新图片必须生成新文件名，不要原地覆盖。
- **文件大小限制**：GitHub 单文件建议低于 50MB，jsDelivr 官方推荐不超过 20MB。大图先压缩再上传。
- **公开仓库无隐私**：图片是公开可访问的，截图如果包含测试数据、内部页面，必须脱敏或裁剪。私有仓库无法走 jsDelivr。
- **国内访问不稳定**：`cdn.jsdelivr.net` 在某些网络环境下可能 DNS 污染或 TLS 握手慢。可以配置 fallback，例如 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`，或回退到 GitHub raw。
- **特殊字符路径**：空格、中文、`#` 会导致 URL 解析问题。文件名只允许 `[a-z0-9-_]`。
- **Git 历史膨胀**：二进制文件频繁提交会让仓库克隆变慢。可以定期归档或另建仓库。
- **权限控制**：上传脚本需要推送权限，建议使用 deploy key 或 fine-grained PAT，只授予 assets 仓库的 Contents 写权限，不要使用全局 PAT。

## 可复用建议

- 封装为 MCP 工具：`upload_image(local_path)` 返回 `{ url, cdn_url, fallback_url }`，方便 Agent 直接调用。
- 每次生成图片后立即上传，并把返回 URL 写入消息卡片，避免只留本地路径。
- 对 URL 做 HEAD 健康检查，如果 CDN 不可达则自动切换 fallback 域名。
- 适合场景：文档配图、测试截图、低流量 OpenClaw 输出物。量级上来后，还是建议迁到对象存储 + 自有 CDN。

## 总结

GitHub + jsDelivr 的最大价值不是“免费”，而是把图床纳入 Git 工作流，可脚本化、可版本化、可自动化。缺点是缓存不可刷新、公开可见、国内节点不稳定。对个人项目和 Agent 实践足够，生产环境需要评估流量、隐私和稳定性要求。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/393924f532bef171.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b0ffcfd77183c1dc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/38925897b7e0ab26.png)

