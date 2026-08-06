---
title: 利用 GitHub + jsDelivr 为自动化流水线搭建免费图床：工程化实践与踩坑记录
feedId: 31879
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么自动化场景需要一个“可控”的图床

使用 OpenClaw、Agent、MCP 等工具构建自动化内容管道时，经常会生成大量图片——可能是 AI 绘图的输出、数据报表的截图，或者是 MCP 工具返回的可视化结果。这些图片需要稳定的公网 URL，才能嵌入到 Markdown 报告、发布到静态博客，或是推送到即时通讯工具中。

常见的第三方图床要么有容量/流量限制，要么随时可能调整服务条款。自建对象存储（S3/R2）虽然稳定，但需要额外维护和潜在的成本。对于一个快速原型或轻量级自动化流程，我们希望有一个零费用、高可用、且能与 GitHub 生态无缝集成的方案。GitHub 仓库 + jsDelivr CDN 就是这样一种组合。

## 核心思路

将图片文件托管在 GitHub 公开仓库中，利用 jsDelivr 的全球 CDN 进行加速。jsDelivr 支持直接从 GitHub 仓库拉取文件，并提供缓存、压缩和就近访问。自动化脚本在生成图片后，通过 Git 或 GitHub API 将文件推送到图床仓库，随即拼接出 jsDelivr URL，插入到后续内容中。

## 工程化步骤

### 1. 创建专用图床仓库

新建一个公开 GitHub 仓库，例如 `assets` 或 `static-images`。保持仓库简洁，内部分目录管理不同来源的图片：

```
repo/
  daily-charts/
  ai-outputs/
  screenshots/
```

仓库本身不宜过大（GitHub 建议单个仓库小于 1GB），单文件需控制在 jsDelivr 的 50MB 限制以内。图床图片通常远小于此，绰绰有余。

### 2. CDN 链接构造规则

图片推送到仓库后，可通过如下规则的 URL 访问：

```
https://cdn.jsdelivr.net/gh/{username}/{repo}@{branch}/{path}
```

例如 `main` 分支下的 `daily-charts/sales.png`：

```
https://cdn.jsdelivr.net/gh/yourname/assets@main/daily-charts/sales.png
```

`@main` 可以锁定分支，也可以使用 commit hash 进一步固定版本。不推荐使用 `@latest`（实际可能解析出错），明确指定分支即可。

### 3. 自动化上传：GitHub Actions 还是外部脚本？

两种方式都可行，取决于你的自动化工具链。

- **在 GitHub Actions 中**：将生成的图片作为构建产物，通过 `actions/checkout` 检出图床仓库，复制文件后执行 `git commit && git push`。这种方法适合完全在 GitHub 生态内的流水线。

- **在外部 Agent 流程中**：使用 Personal Access Token（PAT）调用 GitHub Contents API 写入文件。可用 Node.js、Python 或直接通过 `curl` 实现。OpenClaw 中可以封装成一个 Skill 或直接调用系统命令。

简易 Node.js 上传示例（使用 `@octokit/rest`）：

```javascript
const { Octokit } = require("@octokit/rest");
const octokit = new Octokit({ auth: process.env.GH_TOKEN });

async function uploadImage(owner, repo, path, contentBase64) {
  await octokit.repos.createOrUpdateFileContents({
    owner,
    repo,
    path,
    message: `Upload ${path}`,
    content: contentBase64,
    branch: "main",
  });
}
```

注意：调用 API 需要文件内容 Base64 编码，单个文件大小限制为 1MB（API 限制）。若文件较大，建议走 Git 命令行推送。

### 4. 生成唯一文件名以避免缓存击穿

jsDelivr 对文件进行了长时间缓存。如果更新了同名图片，CDN 边缘节点不会立即刷新。简单的解决方案是每次上传时生成唯一文件名，例如：

```
{date}_{shortuuid}.png
```

这样彻底避免缓存问题，旧图片可以定期清理。自动化脚本可内置 uuid 生成逻辑，绝不依赖固定文件名。

## 踩坑记录

### 仓库公开性与 PAT 权限

图床仓库必须公开，否则 jsDelivr 无法访问。PAT 的权限最小化配置：给予 `Contents` 写权限即可，不要开启不必要的仓库管理权限。Token 应存放在环境变量/密钥管理系统中，避免硬编码。

### API 上传冲突

使用 `createOrUpdateFileContents` 时需提供 `sha` 参数（更新已有文件时）。如果是首次创建，可以不传。但高并发下可能出现竞争条件，导致推送失败。建议在自动化中加入重试机制，或者统一使用命令行 `git pull --rebase && git push`。

### 大小写敏感与路径

jsDelivr 对文件路径大小写敏感，务必与仓库中实际名称完全一致。自动化生成文件名时，保持全小写或统一驼峰，并在拼接 URL 时仔细检查。

### 流量与滥用限制

jsDelivr 免费服务对个人项目足够慷慨，但若短时间内产生大量异常流量，可能被临时限流。对于个人自动化任务（每天几百次请求）不会触及上限，但要避免使用图床托管大体积视频或进行热文件分发。

### 链接防盗链

jsDelivr 不提供内置的防盗链功能，任何拿到 URL 的人都可以直接访问。如果图片包含敏感信息，建议在上传前添加访问令牌逻辑（如 Cloudflare Workers 代理），或换用带鉴权的存储方案。一般自动生成的报表图片没有严格保密要求，风险可控。

## 可复用的集成建议

1. **封装为 MCP 工具或 OpenClaw Skill**：抽象成一个 `upload_to_image_cdn(localPath, remoteDir)` 函数，内部处理 Git 推送或 API 调用，返回完整 CDN 链接。后续所有 Agent 只需要调用这一能力。

2. **配置化仓库与分支**：将 owner、repo、branch 通过环境变量注入，便于在不同项目间切换或复用。

3. **定期清理策略**：设置 GitHub Actions 定时任务，删除 30 天前的旧图片（可通过 Git 历史操作实现），避免仓库无限膨胀。

4. **失败兜底**：上传失败时，将图片本地暂存并发送告警，避免静默丢失关键图片。

5. **CDN URL 缓存本地**：在高频重复引用的场景中，可以将 CDN 链接存入本地 KV 或文件，避免每次重新计算，但不要绕过文件名唯一性。

## 总结

GitHub 仓库 + jsDelivr 的组合为自动化图片托管提供了一种“零成本、可编程、全球分发”的轻量级方案。通过合理设计目录结构、文件名策略和上传管道，可以将它作为 OpenClaw/Agent 工作流中可靠的一环，无需依赖外部服务，也不必担心链接稳定性。当需求增长到需要精细化权限控制或更高 SLA 时，再平滑迁移到专业对象存储，这套图床已经帮你完成了从 0 到 1 的覆盖。

---

