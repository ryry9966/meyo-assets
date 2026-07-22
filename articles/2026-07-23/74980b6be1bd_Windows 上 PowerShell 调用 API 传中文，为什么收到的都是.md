---
title: Windows 上 PowerShell 调用 API 传中文，为什么收到的都是问号？
feedId: 30139
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景：一条中文 JSON 从 Windows 进入 API 时发生了什么

在给 OpenClaw 做 MCP 插件、Agent 自动化脚本时，不少开发者会把 Windows 当作运行环境，用 PowerShell 调用外部 API。只要请求体里不出现中文，一切都顺畅；可一旦接口需要发送中文 JSON，服务端日志里经常出现 `????`、`ÎÒÊÇÖÐÎÄ` 或者直接乱码。更麻烦的是，这个问题在同事的 Mac/Linux 上“没有复现”，于是很容易被归因成“Windows 的锅”。

实际上，出问题的那一段链路很短：**PowerShell 把内存中的 .NET 字符串转换成网络字节流时，用了错误的编码**。而 Windows 上默认的“ANSI 代码页”常常不是 UTF-8，是导致这一错误的根本原因。

面向 OpenClaw/Agent/MCP/插件/自动化实践者，本文会用一个可复现的最小案例说明问题，给出两种从根源解决的方法，并总结可复用的工程化建议。

## 问题：Invoke-RestMethod 的编码陷阱

典型故障场景如下：用 PowerShell 5.1（Windows 自带版本）调用一个接受 JSON 的接口，其中含有中文用户名、备注等字段。

```powershell
$body = @{
    user   = "张三"
    action = "登录"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

在 httpbin 返回的 `data` 中看到的却是 `"user": "??"` 或 `"user": "å¼ ä¸‰"`。即使 Content-Type 里明确写了 `charset=utf-8`，问题依旧。这是因为 `Invoke-RestMethod` 在把字符串 `$body` 转成字节数组时，并没有使用 UTF-8，而是使用了当前会话的 `[System.Text.Encoding]::Default`。在中文 Windows 上，`Default` 通常对应 GBK（代码页 936），于是中文被编码成 GBK，服务端却按 UTF-8 解析，结果自然是乱码。

同样的问题也存在于 `Invoke-WebRequest`，以及我们给 OpenClaw 工具链写的自动化脚本、MCP 插件的本地执行器里。

## 最小复现步骤

如果你想自己验证，可以在 Windows 终端（PowerShell 5.1）执行以下脚本：

```powershell
$body = @{ text = "你好世界" } | ConvertTo-Json
$response = Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $body -ContentType "application/json; charset=utf-8"
$response.data
```

如果输出 `"text": "ä½ å¥½ä¸–ç•Œ"` 或 `"text": "????"` 就说明中招了。

需要注意：如果你用的是 PowerShell 7，这个问题可能不会出现，因为 PowerShell 7 已默认对所有输入输出采用 UTF-8。但大量自动化环境仍运行内置的 Windows PowerShell 5.1，所以排查和根治仍有必要。

## 根源分析：字符到字节的三层选择

PowerShell 处理 HTTP 请求体的编码链大致是：

1. **脚本文件编码**：如果脚本本身是 UTF-8 with BOM，PowerShell 5.1 能正确识别中文字符串（BOM 触发了 UTF-8 解析）。无 BOM 的 UTF-8 会被误判为 ANSI，造成第一步就损坏。
2. **字符串到字节的转换**：`Invoke-RestMethod` 内部调用 `[System.Text.Encoding]::GetBytes(string)`，如果不显式编码，就会用 `[System.Text.Encoding]::Default`。
3. **HTTP 头中的 Content-Type**：这个头只是告知服务端“我用了什么编码”，但不会改变客户端实际发送的字节。也就是说，你声明 UTF-8 但实际发 GBK 字节，服务端当然解析失败。

此外，`$OutputEncoding` 这个变量常被误以为能控制网络编码，其实它只影响 PowerShell 将数据通过管道传给外部程序时的编码，对 `Invoke-RestMethod` 的 Body 无效。

## 做法：两种可靠的修复方法

### 方法一：手动控制字节编码（兼容所有版本）

不要直接传字符串给 `-Body`，而是先用 UTF-8 编码成字节数组，再配合一个 `MemoryStream` 或直接传字节数组给 `Invoke-WebRequest`。注意 **`Invoke-RestMethod` 的 `-Body` 参数只接受字符串，不接受字节数组**，所以这里要用 `Invoke-WebRequest` 或者改用 .NET HttpClient。

```powershell
$body = @{ text = "你好世界" } | ConvertTo-Json
$utf8Body = [System.Text.Encoding]::UTF8.GetBytes($body)

$response = Invoke-WebRequest -Uri "https://httpbin.org/post" -Method Post -Body $utf8Body -ContentType "application/json; charset=utf-8"
($response.Content | ConvertFrom-Json).data
```

这种做法的优点是：编码能被你完全掌控，不受 PowerShell 版本或系统默认代码页影响。缺点是需要把 `Invoke-RestMethod` 换成 `Invoke-WebRequest`，并手动解析响应。

### 方法二：将 `[System.Text.Encoding]::Default` 临时改为 UTF-8（仅 PowerShell 5.1 适用）

通过修改当前进程的默认编码，可以让 `Invoke-RestMethod` 在序列化字符串时走 UTF-8：

```powershell
[System.Text.Encoding]::Default = [System.Text.Encoding]::UTF8
```

再次执行原先的 `Invoke-RestMethod` 调用，中文就会正常发送。但这种方法有副作用：会影响脚本中其它依赖 Default 编码的操作（比如读文件、写日志），务必在脚本结束时还原。因此不太推荐在大型工程中零散使用。

## 踩坑点

- **Content-Type 中 `charset` 不起作用**：很多帖子只告诉你要加上 `charset=utf-8`，但这只解决服务端解析问题，不解决客户端发送的字节错误。
- **`$OutputEncoding` 的误用**：有同事设置 `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)` 后以为万事大吉，但实际上 `Invoke-RestMethod` 并不读取该变量。
- **脚本文件保存格式**：VS Code 默认保存为 UTF-8 无 BOM，PowerShell 5.1 可能将其当 ANSI 读取，导致字符串定义时就坏了。建议在脚本开头加上 `#` 确保文件本是 UTF-8 with BOM，或用 `chcp 65001` 也无济于事，因为那只是控制台代码页。
- **PowerShell ISE 差异**：ISE 环境可能缓存编码设定，导致行为与终端不同。建议直接在独立 pwsh.exe 下测试。

## 可复用的工程化建议

面向 OpenClaw/Agent/MCP 这类长期维护的自动化工具，把上述踩坑转换为几个固定习惯：

1. **封装统一的 HTTP 请求函数**，内部强制使用 `[System.Text.Encoding]::UTF8.GetBytes()` 和 `Invoke-WebRequest`。未来即使迁移到 PowerShell 7 也可保持兼容。
2. **强制脚本文件 UTF-8 with BOM**：在团队规范或 CI 检查中确保 `.ps1` 文件带 BOM，或者一律使用 PowerShell 7 执行。
3. **在关键自动化节点加入编码检测**：执行一次快速自检（向测试接口发送中文并验证回显），确保环境干净。
4. **迁移到 PowerShell 7**：如果你的 OpenClaw 组件运行在 Windows Server 或 CI 上，建议预装 PowerShell 7，从根本上避免 ANSI 陷阱。

## 总结

Windows 上 PowerShell 调用 JSON API 时中文变为乱码，本质是因为 `Invoke-RestMethod` 使用了系统默认的 ANSI 编码将字符串转为字节。简单的 `charset=utf-8` 声明并不能修复字节层面的错误。解决问题的可靠路径是**用 `[System.Text.Encoding]::UTF8.GetBytes()` 手动构建请求体**，并结合 `Invoke-WebRequest` 发送。对于还在用 Windows PowerShell 5.1 的自动化工作流，这是一个值得写到团队知识库里的补丁。

只有理解从脚本文件 -> 字符串 -> 字节流 -> HTTP 的每一层编码决策，才能真正不让中文在 Windows 上被“打坏”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/2c639664bb192ab1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/bc11d7d086f86b46.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/07d9f571b7638ea4.png)

