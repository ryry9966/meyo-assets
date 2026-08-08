---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 32199
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

无论是在 OpenClaw 生态中用 MCP 连接工具，还是通过自定义脚本扩展 Agent 能力，工具声明往往会散落在多个文件里：`mcpServers` 配置写在 JSON 里，实际调用脚本的路径、密钥又藏在 `.env` 或代码深处。本地开发时一切正常，换一台机器或者推到 CI 环境就开始报“找不到命令”“未授权”“端口冲突”。

问题的根源不是 Agent 本身，而是工具配置缺失了“环境差异”这一维度的管理。我们需要的是一种可读、可版本控制、可适配多环境的工具声明方式，同时避免将敏感信息硬编码进配置。`tools.md` 就是这样一种轻量级方案：用 Markdown + YAML 代码块描述工具，通过占位符注入环境变量，再配合加载器在不同环境间无缝切换。

## 问题拆解

Agent 的工具管理通常踩三个坑：

1. **硬编码路径与密钥**：工具脚本路径写死 `/Users/alice/scripts/search.sh`，API key 直接藏在配置里，换环境就失效，还可能泄露。
2. **多环境混战**：开发用的测试库密钥、生产用的正式密钥散落在不同人的本地，没有统一的切环境机制。
3. **可观测性差**：工具声明藏在代码或复杂 JSON 里，新成员接手时看不懂到底有哪些工具、每个工具依赖什么环境变量。

`tools.md` 的思路就是把这几个问题收敛到一个人类优先的文档里，同时保留机器的可解析性。

## 做法：用 Markdown + YAML 块定义工具与环境

### 1. 创建 `tools.md` 并约定格式

文件放在 Agent 项目根目录，每个工具用一个 YAML 代码块表示，环境变量用 `${VAR_NAME}` 占位。

```markdown
# Agent Tools

## search_docs
```yaml
name: search_docs
description: Search internal knowledge base
command: node tools/search.js --base-url ${KB_BASE_URL} --api-key ${KB_API_KEY}
env:
  - KB_BASE_URL
  - KB_API_KEY
```

## deploy_notify
```yaml
name: deploy_notify
description: Send notification to Slack on deploy
command: bash scripts/notify.sh ${SLACK_WEBHOOK_URL}
env:
  - SLACK_WEBHOOK_URL
```

这种格式既可以在 GitHub/GitLab 上直接渲染成漂亮的文档，又能被简单解析器以最小代价提取。

### 2. 编写加载器：解析并注入环境变量

以 Node.js 为例，读取 `tools.md`，正则匹配 YAML 代码块，用 `dotenv` 加载 `.env` 文件后替换占位符。

```javascript
const fs = require('fs');
const yaml = require('js-yaml');
const dotenv = require('dotenv');

dotenv.config(); // 加载 .env，也可根据 NODE_ENV 加载 .env.production 等

const parseToolsMd = (filePath) => {
  const content = fs.readFileSync(filePath, 'utf8');
  const yamlBlocks = content.match(/```yaml\n([\s\S]*?)```/g);
  if (!yamlBlocks) throw new Error('No YAML blocks found in tools.md');
  
  return yamlBlocks.map(block => {
    const yamlStr = block.replace(/```yaml\n|```/g, '');
    const toolDef = yaml.load(yamlStr);
    // 替换所有字符串中的 ${VAR} 为环境变量值，未设置则报错
    const interpolate = (str) => str.replace(/\$\{(\w+)\}/g, (_, varName) => {
      if (!process.env[varName]) throw new Error(`Missing env var: ${varName}`);
      return process.env[varName];
    });
    return {
      ...toolDef,
      command: interpolate(toolDef.command),
    };
  });
};

const tools = parseToolsMd('./tools.md');
```

这样一来，只要保证运行时环境变量到位，同一份 `tools.md` 在任何机器上都能正确执行。

### 3. 管理多环境差异

不要为每个环境维护一份完全独立的 `tools.md`，而是通过环境变量注入差异部分。敏感信息放在 `.env.*` 文件中，并加入 `.gitignore`，只在 CI 或部署时由安全变量注入。

推荐做法：提供 `.env.example` 列出所需变量示例，团队按需复制。

```bash
# .env.example
KB_BASE_URL=https://kb-dev.example.com
KB_API_KEY=your-dev-key
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
```

实际开发中，使用 `direnv` 或 `dotenv-cli` 自动加载对应环境文件，减少手动 `source` 的出错机会。

### 4. 接入 MCP 或 Agent 运行时

如果 Agent 是通过 MCP 服务器调用的，同样可以把 MCP 配置融入 `tools.md`。例如，声明一个 MCP 工具类型的 YAML 块：

```yaml
type: mcp
name: github
command: npx
args: ["-y", "@modelcontextprotocol/server-github"]
env:
  GITHUB_PERSONAL_ACCESS_TOKEN: ${GITHUB_TOKEN}
```

加载器识别 `type: mcp` 后拼装成 MCP 客户端所需的 `mcpServers` 配置，实现同一份声明驱动本地与远程 MCP 服务。

## 踩坑点实录

- **YAML 缩进敏感**：占位符在命令字符串里若包含 `:` 等特殊字符，应加引号避免解析失败，如 `command: "curl -H 'Authorization: ${TOKEN}' ..."`。
- **环境变量丢失**：本地切到 CI 环境时，经常忘设某个 `KB_BASE_URL`。在加载器里对每个引用的变量做严格校验，缺少则启动即中断，而不是等到实际调用时才报 4xx。
- **路径分隔符**：若 command 中直接写 `./scripts/xxx.sh`，在 Windows 下需用 `bash` 包裹或使用 `cross-env`。建议同步提供 POSIX 兼容脚本，并在文档中标注。
- **敏感信息进入版本控制**：偶尔有人直接把 `.env` 或带真实密钥的 YAML 块提交。利用 pre-commit hook 扫描 `${...}` 之外的 ASCII 密钥特征词（如 `-----BEGIN`）做阻断，或使用 `git-secrets`。

## 可复用的工程化建议

1. **保持 tools.md 为声明式清单**，不要在里面写任何业务逻辑。
2. **强制变量校验**：加载器中用 JSON Schema 校验每个工具定义的必要字段，例如必须有 `name`、`command`，`env` 列表需与 command 中的占位符一致。
3. **环境隔离**：用 `.env.development`、`.env.staging` 等文件，配合 `NODE_ENV` 自动选择，CI 平台通过变量注入覆盖。
4. **与 MCP 配置统一**：扩展 `tools.md` 的语法，用一个 `type` 字段区分原生命令工具和 MCP 服务器，从而用一个文件管理 Agent 的全部工具能力。
5. **文档即配置**：利用 Markdown 的元数据（如 YAML front-matter）也能承载配置，但考虑可读性，**代码块方式更直观**，在 PR review 时差异一目了然。
6. **引入 dry-run 模式**：加载器增加 `--dry-run` 参数，只打印替换后的命令，方便调试。

## 总结

`tools.md` 不是银弹，但它把 Agent 工具配置的“文档性”和“可执行性”拉到了同一个平面上。当新同事拿到项目，打开 `tools.md` 就能看到所有可用工具及它们依赖的环境变量；当代码推到生产环境，相同的定义配合注入的密钥就能正常工作。通过严格的变量校验与环境文件模板，团队可以告别“在我这儿明明能跑”的尴尬，让 Agent 的工具链真正做到一次声明，处处运行。

---

