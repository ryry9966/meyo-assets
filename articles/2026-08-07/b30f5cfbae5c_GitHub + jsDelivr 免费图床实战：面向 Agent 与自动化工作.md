---
title: GitHub + jsDelivr 免费图床实战：面向 Agent 与自动化工作流的稳定图片外链方案
feedId: 31942
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在为 OpenClaw 编写插件、MCP 工具或 Agent 自动化流程时，经常需要为生成的图片（如截图、图表、二维码）提供稳定的外链。常见的第三方图床要么有访问频率限制，要么存在外链失效、服务关停的风险；而自建 OSS + CDN 虽然可靠，却增加了额外的成本和运维负担。在个人项目或原型验证阶段，我们更倾向于选择一种零成本、自带版本控制且方便集成到现有 Git 工作流中的方案——GitHub 仓库 + jsDelivr CDN 便成为理想选择。

## 问题分析

直接使用 GitHub raw 链接（`raw.githubusercontent.com`）存在明显问题：国内访问速度慢、缓存不理想，且 GitHub 官方并不推荐把 raw 当作生产环境的 CDN。jsDelivr 是全球性公共 CDN，能够代理 GitHub 仓库的文件，提供快速、带缓存和 HTTPS 的访问链接。把图片存储在 GitHub 仓库中，通过 jsDelivr 分发，就可以得到一个免费、稳定且可控的图床。

但在自动化场景下，简单的“手动上传 → 复制链接”模式难以满足需求。我们需要一套可编程的上传接口，让 Agent 或脚本在生成图片后直接推送到图床并返回 CDN 地址，形成闭环。

## 核心做法与步骤

### 1. 创建专用图床仓库

在 GitHub 新建一个公开仓库（例如 `static-assets`），用于统一存放图片资源。建议采用如下目录结构：

```
static-assets/
└── images/
    ├── 2024/
    │   ├── agent-demo.png
    │   └── error-screenshot.png
    └── ...
```

### 2. 理解 jsDelivr 链接规则

jsDelivr 提供以下两种常用链接格式：

- 基于最新提交：`https://cdn.jsdelivr.net/gh/用户名/仓库名@main/文件路径`
- 基于指定 tag 或 commit：`https://cdn.jsdelivr.net/gh/用户名/仓库名@v1.0/文件路径`

**强烈建议使用 tag 或特定 commit hash**，这样即使后续更新文件，旧的 CDN 链接仍然保持稳定。对于需要长期引用且不希望因内容更新而破坏缓存的场景，这一点至关重要。

### 3. 手动上传与链接生成

简单场景下，可直接通过 Web 界面上传图片，然后按照上述规则拼接链接。也可以借助 PicGo 等工具，配置 GitHub 图床并自动生成 jsDelivr 链接，适合开发者日常使用。

### 4. 自动化上传：通过 GitHub API 实现

面向 Agent 和 MCP 工具的关键在于自动化。以下是一个基于 `@octokit/rest` 的 Node.js 上传示例，将 base64 编码的图片推送到仓库并返回 jsDelivr 链接：

```javascript
const { Octokit } = require("@octokit/rest");
const fs = require("fs");

async function uploadImage(owner, repo, imagePath, base64Content, token) {
  const octokit = new Octokit({ auth: token });
  const path = `images/${imagePath}`;
  const message = `Upload ${imagePath}`;
  
  const { data } = await octokit.repos.createOrUpdateFileContents({
    owner,
    repo,
    path,
    message,
    content: base64Content,
    branch: "main"
  });
  
  const cdnUrl = `https://cdn.jsdelivr.net/gh/${owner}/${repo}@main/${path}`;
  return cdnUrl;
}
```

这里的 `createOrUpdateFileContents` 如果文件已存在会覆盖，需要留意。更稳健的做法是先检查文件是否存在，避免无意覆盖导致旧链接指向新内容。返回的链接可以直接传递给下游 Agent 作为图片 URL。

在实际 MCP 工具开发中，可以将该函数封装为一个 `upload_resource` 工具，接受文件名和 base64 数据，环境变量中读取 `GITHUB_TOKEN`。Agent 在需要图片外链时调用此工具即可完成闭环。

## 踩坑点与注意事项

### 缓存刷新延迟

jsDelivr CDN 在全球有缓存节点。更新仓库图片后，CDN 链接不会立即生效，最长可能等待 12 小时。如果用了 `@main` 引用，可以通过强制刷新缓存接口解决：  
访问 `https://purge.jsdelivr.net/gh/用户名/仓库名@main/文件路径`（注意这是实际 purge 路径的前缀，具体需查文档），或者改用带版本号的 tag，每次更新打一个新 tag，避免缓存问题。  
对于自动化流水线，建议每次上传新文件都使用唯一文件名（如带时间戳或内容哈希），彻底规避缓存陈旧问题。

### 仓库与文件大小限制

GitHub 建议单个仓库保持 1GB 以下，单个文件不超过 100MB（jsDelivr 官方推荐 ≤20MB）。作为图床使用时，一般不会触及这个上限，但需要定期清理无用图片。可以在仓库设置中启用 Actions 定期扫描，删除未被引用的老文件。

### API 速率限制

对 GitHub API 的未认证请求限制为每小时 60 次，使用 Personal Access Token 后提升至 5000 次/小时。自动化工具务必使用 Token，并在多 Agent 并发场景下考虑队列化请求，避免 403。

### 跨域与防盗链

jsDelivr 默认允许所有域跨域访问，适合前端直接使用。如果不需要防盗链，默认配置即可满足；若有防盗链需求，可以考虑通过 Cloudflare Workers 等反代，但会增加额外的复杂度，不属于“免费图床”范畴。

## 可复用建议

1. **建立专用图床仓库并开启 Actions 自动优化**：每次 push 图片时，使用 `sharp` 或 `imagemagick` 压缩 PNG/JPEG，减小体积、加快 CDN 加载。
2. **遵循 Semver 或哈希版本化**：如需稳定引用，推荐用 Git tag 作为版本号，替换 `@main`。自动化工具在上传后自动创建新 tag 或返回新增 commit 的短哈希，让上游引用精确、可追溯。
3. **使用环境变量管理 Token**：无论是 MCP server 还是自动化脚本，都不要硬编码 GitHub Token。利用 `.env` 或 Secrets Manager 注入。
4. **图片命名规范**：`{项目}/{功能}/{日期}-{描述}.{ext}`，方便检索和清理。
5. **监控与兜底**：在 Agent 侧添加图片加载状态检测，若 CDN 链接不可用（极小概率被劫持或屏蔽），可回退到备用图床或 base64 内联展示。

## 总结

GitHub + jsDelivr 方案并不是一个面向高并发生产环境的最优解，但它以极低门槛解决了个人项目、原型验证和自动化流程中“需要外链图片”的核心诉求。配合 GitHub API，我们可以将其无缝嵌入到 OpenClaw Agent 或 MCP 工具链中，实现图片资源的自动存储与分发，让整个工具流更加自洽、可靠。当项目规模增长或稳定性要求变高时，也可以平滑迁移到 OSS + 自有 CDN，因为你的所有图片资产早已通过 Git 实现了版本化治理，迁移成本极低。

---

