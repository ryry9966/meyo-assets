---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的工程实践
feedId: 36811
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

写技术帖、Agent 生成的报告、MCP 插件文档，都绕不开一个需求：图片得有稳定直链。自建图床要维护，商业对象存储要付费（部分还要备案），对低频写作场景都偏重。GitHub 公开仓库 + jsDelivr 的组合是我目前给 OpenClaw 自动化流程选的方案：免费、无备案、只要仓库在链接就长期有效。

## 问题

落地时要同时满足四个约束：

1. 引用链接长期稳定，不因本地路径变化失效；
2. 图片更新后 CDN 能刷到新版本；
3. Agent 流程可全自动上传，不走网页手动点；
4. 大陆访问可达性要提前验证。

## 做法

**第一步：建仓库。** 新建公开仓库 `cdn-assets`，按 `img/yyyy-mm/` 分目录，避免单目录文件爆炸。

**第二步：定 URL 规则。** jsDelivr 的 gh 源格式固定：

```
https://cdn.jsdelivr.net/gh/<user>/cdn-assets@main/img/2025-01/demo.webp
```

`@main` 跟分支，`@commit-sha` 或 `@v1.0` 跟版本。日常写作用 `@main`，要求缓存确定性时锁 commit。

**第三步：脚本化上传。** 我把它封装成 MCP 工具 `upload_image`，核心三步：压缩（sharp 转 webp，控制在 500KB 内）→ 按 `{内容hash8}-{slug}.webp` 命名 → git push。工具返回拼好的 CDN URL，Agent 直接写进 Markdown。多 Agent 并发用 append-only 文件名 + `pull --rebase`，基本无冲突。

**第四步：接入流程。** OpenClaw 生成配图后调 `upload_image` 拿 URL 插入正文，整条链路零人工。

## 踩坑点

- **大陆可达性**：jsDelivr 2022 年后 ICP 失效，大陆直连时好时坏。上线前务必用目标读者网络实测，可备选 `fastly.jsdelivr.net` 或在 `<img>` 上用 `onerror` 回退 raw。
- **缓存刷新**：同名文件覆盖更新后，CDN 可能长期返回旧缓存。最省事的做法是永远用新文件名（hash 前缀），不做覆盖；走 purge 接口也不保证秒级生效。
- **容量上限**：单文件硬上限 50MB；仓库别无限膨胀，GitHub ToS 不欢迎纯存储仓库，超大仓库 jsDelivr 也会限流。老图定期归档到带 tag 的分支。
- **隐私泄露**：公开仓库人人可见。截图带 token、内网 IP 的事故不止一次，上传脚本里加一道敏感词扫描再提交。
- **并发写冲突**：多 Agent 同时 push 偶发 non-fast-forward，统一 rebase + 唯一文件名可规避。

## 可复用建议

- 文件名即版本：`{sha8}-{slug}.webp`，天然规避缓存失效；
- 上传前强制压缩，图床省的是流量不是原片；
- 维护 `manifest.json` 索引，方便 Agent 去重复用；
- 保留回退链：jsDelivr → raw → 自有存储，别把免费方案当 SLA。

## 总结

GitHub + jsDelivr 是典型的"够用但别托付要命场景"的方案：文档、博客、Agent 输出物这类低频访问完全够，零成本且能全自动化接入 MCP 流程；不适合生产级高并发分发。先测可达性，再定命名规范，最后脚本化——三步做完，这条链路就能稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/8bb579cf8d284143.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c92b4ec08c8cfd16.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/fa9e78511e138e3e.png)

