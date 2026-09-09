---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 36775
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

在 OpenClaw 社区写实践帖、跑 Agent 自动化产出图文内容时，图片外链是个绕不开的问题。本地路径贴出去别人看不见，商业图床要么收费要么限制多。我的需求很朴素：免费、稳定、能配合脚本自动上传、URL 可预测。

## 问题

试过几条路：

- 直接贴 GitHub raw 链接：`raw.githubusercontent.com` 在部分网络环境加载慢，偶尔超时；
- 各类免费图床：跑路风险和防盗链，链接随时可能失效；
- 对象存储：要绑域名、配 HTTPS，对个人笔记场景太重。

最后落到 GitHub 公开仓库 + jsDelivr 的组合上。

## 做法

1. 新建一个公开仓库（如 `images`），专门放图；
2. 用 jsDelivr 的 gh 路径引用：
   `https://cdn.jsdelivr.net/gh/<user>/images@main/path/photo.png`；
3. 自动化：把上传流程封装成一个 MCP 工具/脚本，供 Agent 直接调用。流程为：压缩转 webp → 按内容 hash 重命名 → 通过 GitHub API 上传 → 返回 CDN URL。

两个关键习惯：

- `@main` 锁分支，`@<commit>` 锁版本。写正式文档时用 commit hash 保证图不变，日常引用用分支即可；
- 文件名一律用内容 hash，天然规避缓存问题。

## 踩坑点

1. **缓存不更新**：jsDelivr 对同一 URL 缓存很激进，替换同名图片后 CDN 上仍是旧图。解法：hash 命名、新图新 URL；或请求 `purge.jsdelivr.net` 手动清缓存。
2. **默认分支名**：新仓库默认 `main`，老仓库可能是 `master`，路径写错直接 404。
3. **中文文件名**：URL 编码后容易出问题，强制英文+数字命名。
4. **公开性**：仓库公开意味着所有图都能被任何人访问，别传截图里带 token、内网地址的图。
5. **体量边界**：jsDelivr 定位是开源项目 CDN，大量自动上传大图可能触发限制。我的做法是压缩到 200KB 以内、控制单仓库体量，重要内容保留本地原件。

## 可复用建议

- 把上传流程封装成 Agent 可调用的工具（脚本或 MCP server）：入参图片路径，出参 CDN URL，之后所有自动化产出都能直接贴图；
- hash 命名 + commit 锁定，让链接"写了就不变"；
- 图片入库前先压缩，省仓库体积也省 CDN 流量；
- 别把它当唯一存储，本地或另一处留原始副本。

## 总结

GitHub + jsDelivr 不是新方案，但对个人和小团队的图文自动化够用：零成本、URL 稳定、能嵌进 Agent 工作流。它的边界也很清楚——公开性、缓存策略、滥用阈值。想清楚这三点，它就是个省心的默认选项；量级超出个人使用后，再迁移到对象存储也不迟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f6b063a72bc606d1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/30c9b7800df8bc11.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/b540e45f37b7acb5.png)

