---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的工程实践
feedId: 35365
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw、Agent、MCP 插件和自动化工作流里，经常需要返回可公网访问的图片 URL：任务执行截图、插件封面、结果卡片、生成图表等。传统对象存储需要认证和费用，GitHub Raw 访问速度又不稳定。对开源项目或内部小工具来说，GitHub 仓库 + jsDelivr 是一个比较务实的轻量方案。

## 问题

- GitHub Raw 域名在国内和部分网络环境下容易超时或限速。
- 自建 OSS 需要配置权限、密钥，对开源插件不友好。
- MCP 返回图片时，如果没有固定 URL 规则，很难脚本化维护。

jsDelivr 可以缓存 GitHub 公开仓库的静态文件，并提供固定格式的 CDN URL，适合作为免费图床底座。

## 做法/步骤

1. 创建公开仓库，例如 `assets`，目录结构保持简单：

```
assets/
  img/
    202501/
      demo.png
```

2. 上传图片并推送：

```bash
git add img/202501/demo.png
git commit -m "add demo image"
git push origin main
```

3. 通过 jsDelivr 访问：

```txt
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/img/202501/demo.png
```

生产环境更推荐用 tag 固定版本，避免分支更新导致路径内容变化：

```txt
https://cdn.jsdelivr.net/gh/<user>/<repo>@v1.0.0/img/202501/demo.png
```

4. 自动化脚本可封装成一个上传函数：

```bash
#!/usr/bin/env bash
set -e
cd ~/assets
cp "$1" "img/$(date +%Y%m%d)/"
git add .
git commit -m "upload image"
git push origin main
```

在 MCP、OpenClaw 插件或 Agent 中，可以配置一个 `ASSET_BASE_URL`，例如：

```txt
https://cdn.jsdelivr.net/gh/<user>/<repo>@v1
```

之后动态拼接文件路径，避免到处写死完整链接。

## 踩坑点

- **缓存不实时**：jsDelivr 会缓存 GitHub 文件。使用 `@main` 覆盖同名文件时，旧图片可能继续返回。生产环境要么使用 tag，要么每次生成新文件名。必要时可通过 `purge.jsdelivr.net` 触发刷新。
- **仓库必须公开**：私有仓库无法直接通过 jsDelivr 访问。不要放敏感截图、密钥、内部信息。
- **文件大小控制**：GitHub 仓库和 jsDelivr 对单文件大小、仓库体积都有限制。图床场景控制在几百 KB 到 1-2MB 以内比较合理，不要传大视频、PSD 或原始素材。
- **路径大小写敏感**：URL 路径区分大小写，避免空格、中文文件名，减少编码和跨平台问题。
- **不要直接外链 GitHub Raw 再套一层**：直接用 jsDelivr 的 `/gh/` 路径即可。CORS 一般允许前端加载，但不应把它当高并发生产 CDN。

## 可复用建议

- 图片仓库与业务代码分离，降低仓库膨胀风险。
- 目录按年月或内容标识建立，方便后续清理和归档。
- 在 MCP 或 Agent 中封装一个 `assembleAssetUrl()` 辅助函数，统一处理 URL 拼接。
- CI 中上传产物时，配合 GitHub Actions 使用 PAT 提交：

```yaml
- name: Upload result
  run: |
    cp result.png img/$(date +%Y%m%d)/
    git add img
    git commit -m "upload result"
    git push
```

## 总结

GitHub + jsDelivr 适合 OpenClaw、Agent、MCP 插件中的轻量图片托管：免费、有固定 URL 规则、可脚本化、适合开源素材和测试图。核心是公开仓库、tag 控制版本、控制文件大小、理解缓存策略。如果业务要求高可用或隐私保护，应改用对象存储或私有 CDN，不要把这个方案当生产核心依赖。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9d24e72586a2df5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a5848a9a76fdcfbf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d392c88e45d1a032.png)

