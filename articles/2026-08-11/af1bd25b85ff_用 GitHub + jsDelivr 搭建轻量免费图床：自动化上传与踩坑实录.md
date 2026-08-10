---
title: 用 GitHub + jsDelivr 搭建轻量免费图床：自动化上传与踩坑实录
feedId: 32479
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：为何需要一个“可编程”图床

在构建 OpenClaw 可观测性报告、自动化巡检结果、Agent 执行履历等场景时，工作流经常需要输出图表或截图。MCP 工具生成的 PNG 如果能直接获得一个可嵌入 Markdown 的 HTTPS 链接，无论是推送到飞书、钉钉，还是写入 Notion 知识库，都会省掉很多胶水代码。

市面上的免费图床要么不稳定，要么有上传频率限制，甚至悄悄把图片转换成 webp 并压缩，导致截图文字模糊。GitHub 仓库天然就是一个版本化的对象存储，配合 jsDelivr 全球 CDN，可以在保持图片原样的前提下获得稳定、可控的外部访问链接，而且完全免费。对需要在自己控制的管道里“编程式上传”的工程师来说，这是一个很契合的组合。

---

## 问题拆解

我们真正需要的是三个能力：

1. **可脚本化的上传**：不是用 Web 界面上传，而是在 Agent 或 Shell 中直传图片。
2. **固定、缓存友好的 URL**：不同时间上传同一文件名能得到可缓存的地址，且能主动刷新 CDN。
3. **轻型、无额外维护**：不需要搭建 nginx、对象存储，只在 GitHub 仓库中以文件形式管理图片。

GitHub 仓库 + jsDelivr 恰好满足：GitHub API 支持上传文件，jsDelivr 提供 `https://cdn.jsdelivr.net/gh/user/repo@branch/file` 格式的固定链接，并且有缓存清除接口。

---

## 做法与步骤

### 1. 创建公开的图片仓库

jsDelivr 只对公开仓库生效。仓库结构可以简单到：

```
image-host/
  2025/
    screenshot1.png
    chart2.png
```

不要用 `main` 分支频繁增删，建议固定一个分支如 `images`，专门用作图床源，这样可以避免主分支提交记录混乱。存放图片时不推荐用 LFS，因为 jsDelivr 对 LFS 指针文件无效。

### 2. 通过 GitHub API 上传图片

用 Personal Access Token（只需 `repo` 权限）调用 `PUT /repos/{owner}/{repo}/contents/{path}`：

```bash
base64 -w0 screenshot.png > /tmp/encoded.txt
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"add image","content":"'"$(cat /tmp/encoded.txt)"'"}' \
  https://api.github.com/repos/your-username/image-host/contents/2025/screenshot.png
```

注意：API 要求 `content` 为 base64 编码。如果图片较大，建议拆分或使用 Git 大文件流程，但因为图床场景图片通常不超过 5MB，直接 base64 也完全可行。

### 3. 生成 jsDelivr 加速链接

上传后，对应 jsDelivr 链接为：

```
https://cdn.jsdelivr.net/gh/your-username/image-host@images/2025/screenshot.png
```

URL 中的 `@images` 指向分支名。也可用特定 commit hash 如 `@abc1234` 来锁定版本，适合对稳定性要求极高的场景——缺点是每次上传后 commit hash 都会变化，链接无法自动化固定。我们折中选择固定分支名，并用缓存清除机制解决更新问题。

### 4. 自动化集成：用 GitHub Actions 或 OpenClaw 封装

如果图片生成步骤也在 GitHub Actions 中，可以把上面的上传命令写进 workflow，生成后直接推送到图床仓库。对于 OpenClaw Agent，可以把这个上传+返回 CDN 链接的能力封装成 MCP Tool，供 Playwright 截图或 matplotlib 出图后调用。

更轻量的办法是写一个 Shell 函数：

```bash
cdn_upload() {
  local file="$1"
  local dest="2025/$(basename "$file")"
  # encode and upload...
  echo "https://cdn.jsdelivr.net/gh/your-username/image-host@images/$dest"
}
```

Agent 只负责调用命令，并拿到 URL 写入报告。

---

## 踩坑点与排障

- **jsDelivr 缓存刷新不及时**  
  官方声明对 GitHub 源的文件缓存最长 12 小时。新上传或覆盖文件后，CDN 边缘节点可能仍返回旧内容。可使用清除缓存接口：
  ```
  https://purge.jsdelivr.net/gh/your-username/image-host@images/2025/screenshot.png
  ```
  该接口有频率限制，滥用会导致 IP 被封。建议仅在上传脚本末尾对刚修改的文件调用一次。

- **文件名变更 vs 覆盖**  
  直接覆盖相同文件路径，浏览器和中间代理可能因强缓存不重新请求，即便 CDN 已刷新。若图片会频繁更新，最好的方案是文件名带时间戳或内容哈希，这样每个版本都是新 URL，天然绕过缓存。

- **GitHub 上传文件的 size 限制**  
  API 限制单个文件 base64 内容不超过 1MB，但通过 Git LFS 或直接在本地 git push 可以绕过。图床文件通常不超过 2MB，使用 curl 上传绰绰有余。如果图片稍大，可以改成用 `git` 命令直接提交和推送，但脚本复杂度会上升。

- **跨域与 Content-Type**  
  jsDelivr 会自动返回合适的 Content-Type，图片会以 `image/png` 等原样输出，且允许所有源跨域。但在某些严格 CSP 环境中，可能需要为 jsDelivr 添加白名单。

---

## 可复用建议

1. **与环境变量绑定**  
   将 `GITHUB_TOKEN`、仓库名、分支名写在 `.env` 中，让上传脚本零配置可在不同项目中复用。

2. **不要仅依赖 CDN 缓存清除**  
   重要业务场景下，采用“文件名不同就永不冲突”的策略，比如 `image-{timestamp}.png`，这样减少对缓存刷新的依赖，也便于追溯。

3. **搭配 Git 历史回溯**  
   如果图片用于文档历史版本，可以直接切换到对应 commit 的 jsDelivr 链接查看当时的图片，这比传统图床的“覆盖即消失”更安全。

4. **监控和限制**  
   虽然有免费 CDN，但不建议用这个方案承载高流量图片（例如前端页面的大量配图），jsDelivr 对滥用仓库有封禁策略。适合用在低频、制作精良的自动化报告或个人文档中。

---

## 总结

GitHub 仓库 + jsDelivr 搭建的图床特别适合 OpenClaw 这种自动化工具链，它平衡了零成本、可编程性和可控性。主要代价是缓存刷新需要额外小心，以及不宜用于高并发生产环境。对个人和小团队的日常自动化需求而言，几行脚本就能让 Agent 生成的任何图片立刻拥有一个全球可访问的稳定 URL，省去了部署专用图床服务的心智负担。

实践中建议将上传功能做成 MCP Tool，让报告、通知、图表生成的最后一步自动完成图床托管，再串联到后续的消息推送或知识库同步，整个流程就能真正做到“无界面、全自动”。

---

