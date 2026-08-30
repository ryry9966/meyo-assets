---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个够用的免费图床
feedId: 35475
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw、Agent、MCP 或插件的自动化流程里，经常需要把生成的截图、图表、预览图或临时资源发布成一个可公开访问的 URL，供模型读取或外部系统展示。选择图床时，常见的诉求是：可脚本化调用、稳定、免费或低成本、最好还能带版本管理。

第三方图床服务往往有 API 限制、内容审核或收费门槛；自建对象存储要维护服务、配置域名和 HTTPS，对个人项目来说偏重。工程上有一个比较折中的方案：用 GitHub 仓库当存储，用 jsDelivr 做 CDN。这个组合胜在无需运维、接口标准、成本为零，适合小规模自动化和原型验证。

## 问题

直接在 GitHub 里放图片，访问 raw.githubusercontent.com 虽然可行，但国内访问不稳定，而且没有 CDN 加速，偶尔会被限流。jsDelivr 在全球有节点，能免费加速 GitHub 公开仓库里的静态文件，同时支持语义化版本或分支引用。把它接到自动化流程后，Agent 只需要调一个上传函数，就能拿到稳定的外链地址。

不过这个方案不是万能：仓库必须公开，不适合敏感图片；单文件大小和仓库总大小有软限制；jsDelivr 有缓存刷新问题。下面讲具体做法和踩坑点。

## 做法 / 步骤

核心思路：用 GitHub Contents API 上传文件到公开仓库，再拼出 jsDelivr 的 CDN 地址。

1. **创建公开仓库**  
   在 GitHub 新建一个仓库，比如 `assets`，保持 public。初始化一个默认分支（如 `main`）。

2. **生成访问令牌**  
   在 GitHub Settings → Developer settings → Personal access tokens 里生成 fine-grained token，授予该仓库的 Contents 读写权限。把 token 存到环境变量，不要写进代码。

3. **调用 Contents API 上传图片**  
   用任意语言实现一个 `upload_image` 函数。以下是一个 Python 示例：

   ```python
   import requests, base64, os, uuid

   def upload_image(file_path: str, repo_path: str = None) -> str:
       token = os.environ["GITHUB_TOKEN"]
       owner = "yourname"
       repo = "assets"
       branch = "main"

       if repo_path is None:
           ext = os.path.splitext(file_path)[1]
           repo_path = f"images/{uuid.uuid4().hex}{ext}"

       with open(file_path, "rb") as f:
           content = base64.b64encode(f.read()).decode()

       url = f"https://api.github.com/repos/{owner}/{repo}/contents/{repo_path}"
       headers = {
           "Authorization": f"Bearer {token}",
           "Accept": "application/vnd.github+json",
       }
       payload = {
           "message": f"upload {repo_path}",
           "content": content,
           "branch": branch,
       }
       resp = requests.put(url, headers=headers, json=payload)
       resp.raise_for_status()

       return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{repo_path}"
   ```

   如果文件已存在，需要先从 API 获取该文件的 `sha`，再在 PUT 请求中带上 `sha` 字段做更新，否则会返回 422。

4. **接入 Agent / MCP**  
   可以把上面的函数封装成 MCP server 的工具，比如 `upload_image`。OpenClaw 或 Claude 等 Agent 就能在需要时直接调用，把本地截图上传并返回 CDN URL。也可以用 GitHub Actions 在特定事件后自动上传产物，例如测试报告截图。

5. **使用 CDN 地址**  
   返回的地址形如 `https://cdn.jsdelivr.net/gh/owner/assets@main/images/xxxx.png`。在 Markdown、HTML 或 API 响应里直接引用即可。

## 踩坑点

- **仓库必须公开**  
  jsDelivr 只能加速公开仓库的内容。如果图片涉及内部数据，不要用这个方案，换私有对象存储。

- **缓存刷新延迟**  
  jsDelivr 默认缓存 12 小时，覆盖同名文件后，旧内容可能还会被 CDN 节点返回。解决方式：用新文件名（如 UUID）避免覆盖；或者调用 jsDelivr 的 purge API，但有频次限制，不适合高频更新。

- **文件大小限制**  
  GitHub Contents API 单文件上限 100MB，但仓库总大小建议控制在 1GB 以内。jsDelivr 对超大文件支持不好，实践中建议单张图片压缩到 20MB 以下，否则可能无法缓存或访问异常。

- **特殊字符与路径**  
  文件名里的空格、中文、`#`、`?` 等需要做 URL 编码，否则请求会失败或 CDN 找不到资源。建议统一使用英文、数字、连字符和下划线。

- **API 速率限制**  
  GitHub API 对未认证请求限制很严，带 token 后核心限制是 5000 次/小时，但还有二级速率限制。频繁上传时容易被限流，需要加指数退避和重试逻辑。不要在一个循环里无脑 PUT。

- **token 安全**  
  fine-grained token 只授予最小权限，并且只放在服务端环境变量或 secrets 里。不要打包进客户端插件或前端代码。

- **git 历史膨胀**  
  虽然是 API 上传，但每次 commit 都会进入 git 历史。反复覆盖同名文件会让仓库体积增长。建议上传后不再修改，用新文件名代替覆盖。

## 可复用建议

- 把上传逻辑封装成一个 MCP server 或独立 HTTP 服务，供多个 Agent 复用，不要在每个插件里复制代码。
- 文件名使用 `uuid` 或时间戳 + 随机串，避免冲突和缓存问题。
- 上传前先压缩图片：用 Pillow、sharp 或 imagemagick 控制尺寸和格式，比如 PNG 转 WebP。
- 如果担心 jsDelivr 可用性，可以同时返回 raw.githubusercontent.com 地址作为 fallback，由客户端选择。
- 对公开资源建立目录规范，例如 `images/`、`screenshots/`、`diagrams/`，方便后续清理。
- 定期检查仓库大小和 token 权限，必要时归档旧图。

## 总结

GitHub + jsDelivr 是一个适合个人项目、原型验证和小规模自动化场景的免费图床方案。它不需要额外服务器，API 标准化，能快速接入 Agent 工作流。但它有公开性、缓存、大小和速率限制，不适合生产环境或敏感数据。工程上关键是封装好上传函数、控制文件体积、用随机文件名规避缓存问题。把边界想清楚，这个小方案能省掉不少重复造轮子的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a0bbb6a3d8cb5619.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8951c8d91364e44d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6ba12d57544dd7c9.png)

