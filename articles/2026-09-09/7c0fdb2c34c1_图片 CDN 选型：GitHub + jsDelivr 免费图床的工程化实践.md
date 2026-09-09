---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的工程化实践
feedId: 36731
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

写社区帖、项目文档，或者让 Agent 自动产出"图文报告"时，图床是绕不开的一环。可选方案大致三类：商业对象存储（要付费或备案）、第三方免费图床（配额和存续都不可控）、GitHub 仓库 + jsDelivr CDN。第三种零成本、无配额焦虑，且天然适合脚本化，是个人和社区项目性价比最高的起点。

## 问题

直接引用 `raw.githubusercontent.com` 的链接，国内加载成功率偏低；主流免费图床有防盗链、限流和跑路风险。更关键的需求是：上传动作要能被 OpenClaw 的 Agent / MCP 工具直接调用，产完图自动拿回可用的 Markdown 链接，而不是每次手动拖拽。

## 做法

1. 建一个公开仓库，比如 `cdn-images`，专门放图。
2. 生成 fine-grained Personal Access Token，只授予该仓库的 Contents 读写权限。
3. 用 Contents API 上传，全程无界面：

```bash
IMG=$(base64 -w0 ./screenshot.png)
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/cdn-images/contents/img/20250601-a1b2c3.png \
  -d "{\"message\":\"upload\",\"content\":\"$IMG\"}"
```

4. 引用走 jsDelivr：

```
https://cdn.jsdelivr.net/gh/USER/cdn-images@main/img/20250601-a1b2c3.png
```

5. 把第 3、4 步封装成一个 MCP tool：入参本地路径，返回 Markdown 片段，Agent 一行调用完成"上传 + 拿链接"。

## 踩坑点

1. **缓存不可刷新**：jsDelivr 对同名文件缓存很重，覆盖上传后旧链接不会更新。解法是文件名带"日期 + 内容哈希前 8 位"，让每个 URL 天然不可变，不依赖清缓存。
2. **不支持 Git LFS**：仓库开了 LFS，jsDelivr 会 404 或返回指针文本。图床仓库别开 LFS。
3. **国内可达性波动**：2022 年后 jsDelivr 的大陆解析时好时坏。别把它当唯一出口，关键内容保留 raw 链接兜底。
4. **体积限制**：GitHub 单文件硬限 100MB，jsDelivr 代理单文件约 20MB。压缩过的 PNG/WebP 截图完全够用，但别把它当网盘。
5. **token 收敛**：务必 fine-grained + 单仓库，别把全量 classic token 挂在自动化脚本里长期跑。
6. **公开即公开**：任何人都能枚举仓库内容，含敏感信息的截图坚决不进这个仓库。

## 可复用建议

- 文件名统一为 `yyyyMMdd-<hash8>.png`，幂等且免缓存问题。
- 仓库根目录维护一个 `manifest.json`，记录"原文件名 → CDN 链接"映射，将来迁移到 OSS 时脚本可直接消费。
- MCP tool 返回双链接：jsDelivr 为主，raw 为备，下游按可用性降级。
- 上传前本地压缩一次（WebP，质量 80 左右），仓库体积和页面加载都受益。

## 总结

这套方案的定位很清楚：个人博客、社区帖、Agent 自动产出的轻量图床——零成本、可自动化、配合不可变命名后 URL 长期稳定。它不是企业级 CDN，国内可达性要留后手。做好命名规范、权限收敛和双链兜底，运维成本可以概括成四个字：基本不管。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/fd40e211f6076a8a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/70da4e0d8a7a6d46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/06eb9b36cdce3e0c.png)

