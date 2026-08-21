---
title: GitHub + jsDelivr 免费图床实践：给 Agent/MCP 工作流一个可引用的图片 URL
feedId: 34046
source: 综合讨论
publishedAt: 2026-08-21
---

在 OpenClaw、MCP 插件和自动化 agent 实践里，图片经常是中间产物：截图、OCR 标注、生成的架构图、二维码、接口返回的预览图等。很多下游场景只接受 URL 或 Markdown 图片语法，本地路径、`data:` URL 都无法直接用于外部渲染。因此，一个能自动化上传并返回 `https` URL 的图片层是刚需。

免费图床常见问题包括：接口不透明、压缩图片、删除策略不稳定、防盗链、上传限流。对自动化流程不友好。GitHub 仓库作为存储 + jsDelivr 作为 CDN，是相对可控的组合：GitHub 提供标准 API，jsDelivr 提供免费 CDN URL。适合小规模、可公开、非敏感图片。

## 做法 / 步骤

1. **建一个 public 仓库**  
   例如 `assets`，专门放图片。不要放任何敏感信息，因为仓库内容会被 jsDelivr 公开读取。

2. **生成最小权限 token**  
   建议用 GitHub fine-grained token，只授予该仓库的 Contents 读写权限。不要把整个账户的写权限交给脚本。

3. **写上传函数**  
   核心逻辑是调用 GitHub Contents API，将图片 base64 后 `PUT` 到仓库。返回的 jsDelivr URL 格式为：

   ```text
   https://cdn.jsdelivr.net/gh/<owner>/<repo>@<branch>/<path>
   ```

   一个可复用的 Python 片段：

   ```python
   import base64, hashlib, os, requests

   def upload_image(path, repo, branch="main", token=""):
       with open(path, "rb") as f:
           raw = f.read()
       name = hashlib.sha1(raw).hexdigest()[:12] + os.path.splitext(path)[1]
       remote = f"images/{name}"
       url = f"https://api.github.com/repos/{repo}/contents/{remote}"
       resp = requests.put(
           url,
           headers={"Authorization": f"Bearer {token}"},
           json={
               "message": f"upload {name}",
               "branch": branch,
               "content": base64.b64encode(raw).decode(),
           },
       )
       if resp.status_code in (200, 201):
           return f"https://cdn.jsdelivr.net/gh/{repo}@{branch}/{remote}"
       raise RuntimeError(resp.text)
   ```

4. **封装成 MCP 工具或 OpenClaw 插件**  
   暴露 `upload_image(local_path, remote_dir, return_markdown)` 之类的方法，agent 输出时可直接得到 URL 或 `![](url)` 片段，方便粘贴到文档、issue 或聊天窗口。

## 踩坑点

- **缓存**  
  jsDelivr 对默认分支 URL 可能缓存 12–24 小时。覆盖同名文件不会立即生效。解决办法：文件名用内容哈希；或者 URL 中带 commit hash，例如 `https://cdn.jsdelivr.net/gh/user/repo@<commit>/images/foo.png`，可以绕开分支缓存。

- **仓库必须 public**  
  private 仓库 jsDelivr 无法读取。如果图片含内部信息，不要走这个方案。

- **大小限制**  
  GitHub 单文件理论上限 100MB，但 jsDelivr 对 GitHub 文件建议不超过 20–50MB，否则可能加载失败。大图应先行压缩或降采样。

- **可用性波动**  
  GitHub raw 和 jsDelivr 在大陆不同地区表现有波动。不要把它当作关键业务的生产图床。可以在返回 URL 的同时，将图片同步备份到对象存储或本地索引。

- **仓库膨胀**  
  频繁上传截图会让 git 历史快速增长。建议单独使用图床仓库，不要混入代码仓库；定期归档旧目录。

## 可复用建议

- 文件名使用 `sha1` 前 12 位 + 扩展名，天然去重，避免 CDN 缓存和重复上传。
- 上传脚本做成幂等：先检查远程文件是否存在，存在则直接返回 URL。
- Agent 输出时同时返回 Markdown 片段，减少人工拼 URL。
- token 和仓库信息通过环境变量注入，不要硬编码在前端或插件配置里。
- 维护一个 `manifest.json`，记录 `path -> url`，便于检索、清理和后续迁移。

## 总结

GitHub + jsDelivr 免费图床很适合自动化流程中的“图片暂存和分享层”：API 干净、URL 规则明确，配合哈希命名和版本化 URL 可以基本避开缓存坑。但它不是生产图床，公开性、大小限制和国内可达性决定了它只适合非敏感、非关键图片。作为 Agent/MCP 工作流里的默认图片出口，这个组合足够务实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5d64595715fe1bad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/c171942cb458b015.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ba7d0a816c9d201c.png)

