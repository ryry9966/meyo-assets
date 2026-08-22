---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床在 Agent 工作流里的工程化实践
feedId: 34195
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化项目里，经常需要把截图、OCR 结果、插件生成的图表或告警现场转成可公网访问的图片 URL。典型场景包括：错误现场留存、每日报表卡片、插件预览图、通知消息里的配图。

本地路径或临时上传服务不适合自动化，因为 URL 不稳定、接口不统一。我选择 GitHub 公共仓库 + jsDelivr CDN，不是单纯因为免费，而是因为它的 API 可编程、URL 规则可预测，并且能直接和 GitHub Actions、MCP 工具链拼接。

## 问题

直接使用 GitHub raw 链接有几个明显问题：

- `raw.githubusercontent.com` 在国内访问不稳定，部分环境超时。
- raw 链接没有 CDN 缓存优化，大图加载较慢。
- URL 带 commit hash 时模板化困难，每次上传都要替换。
- GitHub 仓库长期存放大量二进制文件会导致体积膨胀，影响 clone 和日常维护。

jsDelivr 能缓解前两点，但引入新问题：缓存刷新不直观、版本规则需要明确、私有仓库不可用、文件大小受限。这些边界需要在自动化设计阶段就写清楚。

## 做法 / 步骤

1. **创建专用 public 仓库**，例如 `cdn-assets`，默认分支设为 `main`。不要放任何敏感图片，public 意味着内容可被遍历。

2. **生成 fine-grained token**：只授予该仓库的 Contents 读写权限，不要使用全局 PAT，避免权限过大。

3. **用 GitHub Contents API 上传**。示例脚本如下：

```bash
REPO="user/cdn-assets"
BRANCH="main"
FILE="img/$(date +%Y%m%d)-$(uuidgen | cut -c1-8).png"
BASE64=$(base64 -w0 "$1")

curl -X PUT \
  -H "Authorization: token $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$REPO/contents/$FILE" \
  -d "{\"message\":\"upload image\",\"branch\":\"$BRANCH\",\"content\":\"$BASE64\"}"
```

4. **拼接 jsDelivr URL**，规则固定为：

`https://cdn.jsdelivr.net/gh/user/repo@branch/path`

例如：

`https://cdn.jsdelivr.net/gh/user/cdn-assets@main/img/20250401-abc123.png`

注意 `@main` 不要省略，否则默认分支变更时行为可能不符合预期。

5. **在 agent/MCP 层封装成工具**。实现一个 `upload_image` MCP tool，输入本地路径或 base64，输出 jsDelivr URL。OpenClaw 插件只需要调用该工具，不用关心 GitHub API 细节。

6. **文件名使用内容 hash 或时间戳 + 随机串**，避免同名文件缓存冲突。内容 hash 更可控：同一张图重复上传会得到同一 URL，不会产生冗余提交。

## 踩坑点

- **文件大小限制**：jsDelivr 对 GitHub 单文件有 50MB 限制，GitHub API 单文件 content 上限约 100MB，但实际超过 20MB 就应该考虑对象存储。截图和图表通常在 1–5MB，问题不大。
- **缓存刷新**：修改同一路径文件，jsDelivr 不会立即更新。如果业务允许，永远用新文件名；如果必须覆盖，调用 purge API 或使用带版本号路径。
- **重名冲突**：GitHub Contents API 对已存在路径要求提供 `sha`，直接 PUT 会返回 422。自动化上传服务里要么先 GET 路径获取 sha，要么文件名直接带内容 hash，将冲突率降到最低。
- **私有仓库不可用**：jsDelivr 无法访问私有仓库，上传后不能通过 CDN 读取。如果图片含内部信息，要走对象存储签名 URL。
- **token 泄漏**：不要把 token 写进前端插件或客户端，只在后端 / MCP server 环境变量中使用。
- **仓库膨胀**：GitHub 建议仓库保持 1GB 以下，超过 5GB 会警告。低频小图没问题，高频大图别硬塞。
- **国内可用性**：jsDelivr 国内大部分可用，但偶尔 DNS 污染。重要链路加超时和 fallback。

## 可复用建议

- 把上传逻辑做成一个独立的 MCP server 或 CLI，不要在每个插件里复制粘贴脚本。
- 图片命名规范：`yyyyMMdd/` 目录 + 8 位 hash + 扩展名，便于清理和排查。
- 上传前做 MIME / 大小校验，只允许 png、jpg、webp，设置 10MB 上限。
- 同时返回 GitHub raw URL 和 jsDelivr URL，debug 时 raw 更直接，业务使用 CDN。
- 必须覆盖同一路径时，使用 `https://purge.jsdelivr.net/gh/user/repo@main/path` 主动刷新缓存。
- 如果上传频率高或图片超过 10MB，直接上 Cloudflare R2 / Backblaze B2 + 自定义域名，jsDelivr 方案只适合中低频、小文件、内部工具。

## 总结

GitHub + jsDelivr 免费图床在 agent / 自动化场景里是务实选择：API 齐全、URL 规则简单、适合脚本和 MCP 封装。但它的边界也很清楚：低频、小图、public 内容、能接受缓存延迟。把边界写进工具说明里，比盲目替换对象存储更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/85ec0410c5b4902c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/bb495f1563b71239.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/0a3f2ba62d5f7674.png)

