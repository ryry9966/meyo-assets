---
title: PowerShell 处理 JSON 中文的编码陷阱与根治方案
feedId: 30665
source: 综合讨论
publishedAt: 2026-07-27
---

# PowerShell 处理 JSON 中文的编码陷阱与根治方案

## 背景：一次“完美”的自动化为何变成乱码生成器

我在维护一套基于 OpenClaw 的自动化运维监控插件时，需要从 Windows 主机上的 PowerShell 脚本定时调用内部 API，抓取设备状态的 JSON 数据并落盘供 Agent 消费。API 返回的字段里包含设备名称、位置、负责人等中文信息。流程很简单：`Invoke-RestMethod` 取回数据，再用 `Out-File` 写入 `.json` 文件。

测试时一切都正常，直到有同事发现：所有中文全变成类似 `ç'–ç»` 这样的乱码，而英文与数字却毫发无伤。这意味着 Agent 后续解析到的全都是无效数据，整个监控链中断。

如果你也在 Windows 上用 PowerShell 对接 OpenClaw、MCP 服务、自动化插件，下面的分析能帮你一次性根除这类编码痼疾。

## 症状与直接原因：多个编码层各说各话

把问题拆开，我们面对的是三层编码：

1. **HTTP 响应体**：绝大多数现代 JSON API 返回的是 `UTF-8` 编码的字节流，Content-Type 里通常会声明 `charset=utf-8`。
2. **PowerShell 的字符串处理**：`.NET` 底层字符串全部是 UTF-16 内部表示，但在与外部字节流交互时，需要指定转换编码。
3. **控制台 / 文件输出**：Windows PowerShell 5.1 的默认输出编码是 `UTF-16LE`（Unicode），而控制台的输出编码（`[Console]::OutputEncoding`）默认跟随系统代码页，比如简体中文是 `GBK`（代码页 936）。

当你随手写下：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/device" -Method Get
$resp | ConvertTo-Json -Depth 5 | Out-File -FilePath "devices.json"
```

看似简单，实际发生了多次错误的编码转换：

- `Invoke-RestMethod` 内部将响应字节流解码为字符串时，可能因为某些头信息缺失或解析错误，默认采用了 `ISO-8859-1`，而不是 `UTF-8`，导致中文字节被按单字节错误映射为不可见字符，再转成 .NET 字符串时已经损坏。
- 即使第一步未损坏，`Out-File` 默认使用 `Unicode`（UTF-16LE）写入文件，而你的后续工具或日志分析系统大概率期望 `UTF-8`。从 `.NET 字符串（UTF-16）` 到 `UTF-16LE` 没有信息丢失，但若目标系统强行用 `UTF-8` 读取，就会再次乱码。
- 如果中间经过控制台输出（如 `Write-Host`），`[Console]::OutputEncoding` 为 `GBK`，会将超出 GBK 范围的字符替换为 `?`，信息彻底丢失。

## 排查与根治步骤

### 1. 确认响应字节流没有被提前破坏

不要直接使用 `Invoke-RestMethod` 的解析结果，改用 `Invoke-WebRequest` 获取原始字节流，手动指定编码解码：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/device" -Method Get
$reader   = [System.IO.StreamReader]::new(
                $response.RawContentStream,
                [System.Text.Encoding]::UTF8
            )
$rawJson  = $reader.ReadToEnd()
$reader.Close()
```

此时 `$rawJson` 是原始 UTF-8 解码后的字符串，中文不会出错。如果 API 明确使用其他编码，可替换对应的 Encoding 对象。

### 2. 显式控制文件输出编码

放弃 `Out-File` 的默认行为，直接使用 `Set-Content` 或 `Out-File` 的 `-Encoding` 参数（Windows PowerShell 5.1 支持 `utf8`，会带 BOM；PowerShell 7+ 推荐 `utf8NoBOM`）。

```powershell
$rawJson | Set-Content -Path "devices.json" -Encoding UTF8
```

或者如果需要加工成对象再序列化：

```powershell
$data = $rawJson | ConvertFrom-Json
# 处理数据...
$data | ConvertTo-Json -Depth 5 | Set-Content -Path "devices.json" -Encoding UTF8
```

### 3. 强制整个会话的编码一致性

在脚本开头加入：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding           = [System.Text.Encoding]::UTF8
```

- `$OutputEncoding` 影响 `|` 管道将字符串传给外部程序时的编码（如 `python` 等）。
- `[Console]::OutputEncoding` 控制控制台输出的编码，减少 `Write-Output` 造成的字符转换丢失。

### 4. 保底方案：使用 PowerShell Core 7+

所有上述问题在 Windows PowerShell 5.1 上尤为突出，而 PowerShell Core 7+ 已将默认编码全面统一为 `UTF-8`，`Out-File` 默认就是 `UTF-8`（无 BOM），且不再依赖系统代码页。如果环境允许，直接升级到 PowerShell 7 是最彻底的解决方案。

## 踩坑点梳理

- **Invoke-RestMethod 自动解析的陷阱**：它返回的对象属性可能已经受损，因为其用了错误编码解码 JSON。看到乱码时，直接从 `RawContentStream` 入手，别再相信 `.Content` 属性。
- **ConvertTo-Json 不会帮你修复错误**：如果输入字符串本身就是损坏的，序列化只是把错误编码后的 .NET 字符串原样输出，不会恢复原始中文。
- **BOM 带来的隐形兼容问题**：某些 JSON 解析器（如 Python 的 `json.load`）在遇到 BOM 时表现不同。建议在需要兼容性时去掉 BOM，使用 `-Encoding utf8NoBOM`（PS7），或在 PS5.1 中用 .NET `[System.IO.File]::WriteAllText(path, content, [System.Text.UTF8Encoding]::new($false))`。
- **脚本文件本身的编码**：`powershell.exe` 执行 `.ps1` 时，如果脚本中含中文，文件另存为 `UTF-8 with BOM` 是最稳妥的选择，否则可能出现“字符串缺少结束符”等解析错误。

## 可复用的工程化建议

- **统一读写工具**：封装一个 `Write-Utf8File` 函数，所有脚本都通过它写文件，避免散落 `Out-File` 裸调用。
- **入口检查**：任何处理 REST API 的脚本，头几行强制设置编码变量，并执行一次 `chcp 65001` 临时改变控制台代码页（虽然作用有限）。
- **测试用例**：在 CI 里加入一条中文数据（如 `{"name":"测试设备"}`）的 API 模拟，每次脚本改动后验证输出的十六进制字节流是否严格符合 UTF-8 编码的中文字节序列。
- **跨平台迁移策略**：如果插件可能从 Windows 迁移到 Linux，直接以 PowerShell Core 7 为目标开发，省去 99% 的编码问题。

## 总结

PowerShell 在 Windows 上的中文 JSON 损坏，本质上是历史遗留的编码债务在自动化流程中的集中爆发。它不是黑魔法，而是一系列可追溯的字节-字符串边界失控。只要养成从原始字节流控制编码、显式指定文件输出格式、升级到统一 UTF-8 的运行环境这三个习惯，中文 JSON API 调用就不会再成为 Agent 的“毒数据”源头。

对于 OpenClaw 社区的自动化开发者来说，这一环也是确保 MCP 工具链和 Agent 插件数据可靠性的基础工程实践。下次见到乱码，不要靠“感觉”去改代码，直接检查每个节点的 byte→string 和 string→file 两个关口，问题很快就能定位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/08d6b66586a128e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/a6c8ed5c09d9dee5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/4b56685f0ab87703.png)

