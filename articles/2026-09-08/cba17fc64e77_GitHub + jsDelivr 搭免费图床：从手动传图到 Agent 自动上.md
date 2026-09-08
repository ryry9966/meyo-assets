---
title: GitHub + jsDelivr 搭免费图床：从手动传图到 Agent 自动上传的实践
feedId: 36580
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

写文档、发博客、跑 Agent 生成带截图的报告，都绕不开“图放哪”这个问题。国内对象存储要备案实名，商用图床按流量计费，而个人项目流量小、预算为零。GitHub 仓库 + jsDelivr 是一个务实的折中：仓库免费，jsDelivr 作为公共 CDN 直接回源 GitHub，相当于给仓库内容加了一层全球缓存。

## 问题

直接用 GitHub 原生方案有三个痛点：

- `raw.githubusercontent.com` 国内访问不稳定，且没有 CDN 缓存；
- 链接一旦硬编码，迁移成本高；
- 手动传图会打断写作和自动化流程。

## 做法

**1. 建仓**：新建一个公开仓库（如 `assets`），按 `images/2025/06/` 目录组织。

**2. 拼 URL**：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>/<file>
```

**3. 命名即缓存策略**：文件名用内容 hash（如 `a3f9c2.webp`）。内容不变 URL 不变，内容变了 URL 必变，天然规避缓存不刷新的问题。

**4. 接入 OpenClaw 自动化**：把它封装成一个 MCP tool（或直接用 GitHub 插件），输入本地图片路径，内部流程是：压缩转 WebP → 算 hash 命名 → 调 GitHub Contents API 上传（需要 PAT）→ 拼出 jsDelivr URL 返回。之后 Agent 写 Markdown 时可直接内联调用。上传核心就是一次 PUT：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<user>/<repo>/contents/images/2025/06/a3f9c2.webp \
  -d '{"message":"upload","content":"<base64>"}'
```

**5. 验证**：`curl -I` 看响应头里的 `cf-cache-status`，确认命中 CDN 而不是回源。

## 踩坑点

1. **同名覆盖不刷新缓存**。jsDelivr 对 `@main` 分支引用也有缓存，覆盖同名文件后旧图可能继续服务数小时。解法：只用 hash 命名、永不覆盖，或 URL 改用 commit hash（`@<sha>`）。
2. **体积限制**。GitHub 拒绝 100MB 以上文件，jsDelivr 对大文件（约 20MB 以上）可能拒绝回源。图床场景建议单图压到 500KB 以内。
3. **国内可用性会波动**。jsDelivr 在国内有过解析异常的历史，不能当唯一依赖。重要文档保留 raw 链接兜底，或前端做 onerror 回退。
4. **滥用风控**。大流量热链接可能触发风控导致仓库被暂停。个人博客量级没事，别拿它当视频床。
5. **仓库必须公开**，所有图等于公开资源，截图里别带 token、内网地址。
6. **文件名避免中文和空格**，URL 编码是这类图床最常见的事故来源。

## 可复用建议

- 把“压缩 → hash → 上传 → 返回 URL”做成独立 MCP tool，任何 Agent 会话都能复用，不要写死在某条工作流里；
- 上传成功后本地追加一份 `manifest.json`（hash、尺寸、URL、时间），方便盘点和迁移；
- 截图先等比缩到宽 1600px 再传，仓库体积可控；
- 上传脚本做幂等：先 GET 检查 hash 是否已存在，避免重复上传烧 API 配额（认证后 5000 次/小时，够用但省着点）。

## 总结

GitHub + jsDelivr 适合个人文档、博客、Agent 产出物这类中小流量场景：零成本、免运维，在 hash 命名的前提下链接足够稳定。边界也清楚——它不是 SLA 服务，国内可达性看运气，不适合商业生产环境。把它当“带缓存的开源网盘”用，同时保留一套随时可迁走的 manifest，是这套方案最健康的姿势。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/f16527b46fdb977e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/4521e646e4d36ad0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/ca7251de946972dd.png)

