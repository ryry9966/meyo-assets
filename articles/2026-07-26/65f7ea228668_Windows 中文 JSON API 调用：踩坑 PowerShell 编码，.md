---
title: Windows 中文 JSON API 调用：踩坑 PowerShell 编码，让你的 Agent 数据别再烂中乱码
feedId: 30513
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：Agent 工具链里最不起眼的「毛细血管」问题

很多自动化实践会在 Windows 上直接用 PowerShell 调用各种 HTTP API——尤其是 MCP 工具、OpenClaw 插件或自建 Agent 的胶水脚本。收到 JSON 后解析、提取中文内容、写入数据库或转发到消息通道，看起来再平常不过。

但只要你接触过中文，大概率遇到这种诡异输出：

- API 返回的中文在终端里显示为 `????`
- 保存到文件后用记事本打开正常，Notepad++ 里却乱码
- `ConvertFrom-Json` 解析后的对象，中文属性值变成 `ä½ å¥½` 之类的乱码
- 从剪贴板粘贴中文到 PowerShell 脚本，执行后变成乱码

这类问题 90% 可归结为：**PowerShell 的默认字符编码处理，在 Windows 中文环境下和 UTF-8 API 返回数据产生了错配**。

下面我会把问题拆解、最小化复现，并给出可落地的工程化处理方式，让 Agent 的胶水脚本不再被编码背刺。

## 问题本质：三层编码不一致

一个典型的 `Invoke-WebRequest` 或 `Invoke-RestMethod` 调用流程，数据在以下三处可能发生编码变化：

1. **HTTP 响应体的字节流 → 字符串**：受 `-ContentType` 或响应头 `charset` 影响
2. **字符串在 PowerShell 进程内处理**：依赖于 `[System.Text.Encoding]::Default` 和 `$OutputEncoding`
3. **输出到文件 / 终端 / 管道**：受 BOM、` Out-File -Encoding`、终端的代码页 (codepage) 控制

Windows 中文版的默认系统编码通常是 **GBK (CP936)**，而绝大部分现代 API 返回 **UTF-8** 数据。  
PowerShell 5.1 的 `Invoke-RestMethod` 在一些情况下会错误地用 `ISO-8859-1` 解码响应字节，导致中文直接被不可逆地损坏。即便 PowerShell 7 已经缓解了部分问题，但如果输出重定向或写到文件时未明确指定 UTF-8，仍然会乱码。

## 最小复现：让人头皮发麻的乱码

假设你有一个返回中文 JSON 的 API（这里用测试服务模拟）：

```powershell
# PowerShell 5.1 中文版 Windows 10/11
$response = Invoke-RestMethod -Uri "http://httpbin.org/anything" -Method POST -Body '{"city":"北京"}'
$response.json.city   # 期望 "北京"，结果可能是乱码
```

或者更直接的：

```powershell
$json = '{"message":"你好世界"}'
$obj = $json | ConvertFrom-Json
$obj.message          # 看起来正常？但如果来自文件读取就可能坏了
```

更隐蔽的是：用 `Invoke-WebRequest` 拿到 Content 后手动解码，如果不指定编码：

```powershell
$r = Invoke-WebRequest -Uri "https://api.example.com/data"
$jsonString = $r.Content    # 已经损坏的字符串（在 PS5.1 中常见）
```

**为什么终端显示正常，一赋值就乱？**  
因为终端使用 GBK 渲染，碰巧字节流被错误解释后又被终端按 GBK 显示回来，肉眼有时看不出来，直到你对字符串做操作，比如传递给其他 API 或写入数据库，才会暴露。

## 根治步骤：在 PowerShell 中安全处理中文 JSON

### 1. 直接拿到原始字节，自行解码

最稳妥的办法是不让 PowerShell 自动推导编码，自己用 UTF-8 解码响应体字节：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -ContentType "application/json; charset=utf-8"
# 从 RawContentStream 读字节
$stream = $response.RawContentStream
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
$jsonString = $reader.ReadToEnd()
$obj = $jsonString | ConvertFrom-Json
$reader.Close()
```

或者更简洁地用 `Invoke-RestMethod` 传递 `-ContentType`，并结合 `-OutFile` 直接保存到文件避免内存编码问题，但多数 Agent 管道需要立即解析，故建议自行处理字节。

### 2. 设定整个会话的输出编码为 UTF-8

在脚本顶部添加：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

- `[Console]::OutputEncoding` 控制控制台输出编码
- `$OutputEncoding` 控制 PowerShell 向外部程序管道传送数据时的编码

**注意**：这只对输出有影响，对 `Invoke-RestMethod` 内部解码没有直接帮助，但能保证你在终端打印或传给其他外部程序时不再乱码。

### 3. 写入文件时强制指定 UTF-8 without BOM

```powershell
$jsonString | Out-File -FilePath "result.json" -Encoding utf8NoBOM
# 或者用 Set-Content -Encoding utf8NoBOM 但在 PS5.1 中只支持 utf8（带BOM）
# 建议用 [System.IO.File]::WriteAllText("result.json", $jsonString, [System.Text.UTF8Encoding]::new($false))
```

### 4. 读取有中文的文件时指定编码

```powershell
$json = Get-Content -Path "config.json" -Raw -Encoding UTF8
$obj = $json | ConvertFrom-Json
```

若脚本需要在多台机器上运行，不要依赖 `-Encoding Default`，显式指定 UTF8 是唯一安全的。

### 5. 跨平台与 PowerShell 版本差异处理

在你脚本中做条件判断，统一 UTF-8 行为：

```powershell
if ($PSVersionTable.PSVersion.Major -ge 7) {
    # PS7 默认 UTF-8 更友好，但仍要小心
    $PSDefaultParameterValues['*:Encoding'] = 'utf8'
} else {
    # PS5.1 需要更严格的字节控制
    [Console]::OutputEncoding = [Text.Encoding]::UTF8
    $OutputEncoding = [Text.Encoding]::UTF8
}
```

## 踩坑实录：这些细节最容易忽略

- **`Invoke-RestMethod` 的 `-ContentType` 参数并不会解决编码问题**：它设置的是请求头，告诉服务器客户端希望接受什么类型，**不影响 PowerShell 如何解码响应体**。
- **PS5.1 的 `Out-File -Encoding utf8` 会写入 BOM**，某些平台解析（如 Node.js 的严格 JSON 解析）可能报错，务必使用 `utf8NoBOM` 或 .NET 方法。
- **管道传递到外部 CLI 工具**时，`$OutputEncoding` 决定传给 stdin 的文本编码。如果外部工具期望 UTF-8 但收到 GBK，同样会乱码。所以在调用 `curl`、`jq` 等前要设置好编码。
- **BOM 导致的 JSON 解析失败隐蔽**：第一行开头出现不可见字符 `\ufeff`，`ConvertFrom-Json` 会直接报错，报错信息不直观，往往让人怀疑数据本身。

## 面向 Agent/MCP 脚本的实战建议

在构建 OpenClaw 插件或 MCP 工具胶水脚本时，请遵守以下规则，可避免 99% 的中文编码问题：

1. **在脚本头部固定编码设置块**（可单独存为 `encoding_init.ps1` 共用）。
2. **所有 HTTP 请求使用原始字节解码**，或者统一使用封装函数 `Invoke-API`，内部采用 `StreamReader` + UTF-8 解码。
3. **JSON 读写一律使用 .NET 类**：`[System.IO.File]::WriteAllText()` 和 `[System.IO.File]::ReadAllText()` 配合 `UTF8Encoding` 无 BOM 实例。
4. **在 CI/CD 或 Agent 运行环境中设定环境变量**：`$env:LC_ALL='C.UTF-8'` 对 PowerShell 无直接影响，但可提醒维护者已进入 UTF-8 模式。
5. **加入编码健康自检**：脚本启动时执行 `ping` 性质的 JSON 往返（构造中文片段 → 保存 → 读取 → 校验），若乱码直接抛异常，防止后续数据污染。

## 总结

Windows 中文环境下 PowerShell 处理 JSON 乱码，本质是系统默认编码（GBK）与现代 API 通用编码（UTF-8）之间的冲突。旧版 PowerShell 5.1 的自动解码机制经常做出错误假设，即便在 PS7 中，文件 I/O 和管道输出仍然需要显式指定 UTF-8。

对于 Agent 和自动化工具链，这种问题往往在部署后才暴露，排查成本高。用一套固定范式——**始终显式处理字节并用 UTF-8 解码，文件读写指定无 BOM 的 UTF-8**——就能彻底告别中文被「打坏」的噩梦。

不要让编码问题吃掉你宝贵的调试时间。把它当作胶水脚本的基建，一次性配好，日后中文 JSON 才能安然穿过每道管线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/45516b97ae2eefd7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/75d40eefc4ad6edb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/8b6c0d8f7d70ae53.png)

