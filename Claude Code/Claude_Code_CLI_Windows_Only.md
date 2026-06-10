# Claude Code CLI

------

## 一、Claude Code CLI 是什么

Claude Code 是 Anthropic 的命令行 AI 编程助手，可以在终端里读取项目、修改代码、运行命令、做代码审查、生成提交、管理插件和 MCP 工具。它既可以交互式使用，也可以用 `claude -p` 做脚本化调用。官方 CLI 支持启动会话、管道输入、继续/恢复会话、更新、MCP、插件等命令。([Claude Code](https://code.claude.com/docs/en/cli-reference))

------

## 二、Windows 安装 Claude Code CLI

### 2.1 推荐安装方式：PowerShell 7 原生安装

#### 2.1.1 安装 PowerShell 7

1. 点击 [PowerShell 7 MSI 安装包](https://learn.microsoft.com/en-us/powershell/scripting/install/install-powershell-on-windows?view=powershell-7.6#msi) 下载安装包，不推荐修改安装路径，修改后 `Windows Terminal` 无法自动识别，需手动添加，较为麻烦。
2. 在微软应用商店安装 `Windows Terminal`，打开 **设置** - **启动** - **默认配置文件**，选择 `PowerShell`，注意，不是 `Windows PowerShell`。

#### 2.1.2 安装 Claude Code CLI

Win + R 打开运行窗口，输入 pwsh 通过终端默认打开 `PowerShell`，输入

```powershell
irm https://claude.ai/install.ps1 | iex
```

安装后验证：

```powershell
claude --version
claude doctor
```

如果 PowerShell 执行策略阻止脚本运行，可以先执行：

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

然后重新运行安装命令。

------

## 三、登录与启动

如果使用 Anthropic 官方 Claude：

```powershell
claude auth login
```

进入项目目录启动：

```powershell
cd C:\path\to\your-project
claude
```

也可以带初始问题启动：

```powershell
claude "请阅读这个项目，告诉我启动、测试、构建命令"
```

------

## 四、Windows 接入 DeepSeek

DeepSeek 提供兼容 Anthropic API 的接口，Claude Code 可以通过环境变量把请求转到 DeepSeek。

DeepSeek Anthropic 兼容地址：

```text
https://api.deepseek.com/anthropic
```

DeepSeek 当前模型包括 `deepseek-v4-flash`、`deepseek-v4-pro`。([DeepSeek API Docs](https://api-docs.deepseek.com/))

### 4.1 持久化配置：环境变量

对当前 Windows 用户长期生效：

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/anthropic", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "<你的 DeepSeek API Key>", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "deepseek-v4-flash", "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", "deepseek-v4-flash", "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", "max", "User")
```

执行后关闭并重新打开 PowerShell。

### 4.2 持久化配置：配置文件

找到 `%USERPROFILE%/.claude/settings.json`，打开并编辑：

```json
{
    "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "<你的 DeepSeek API Key>",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-pro[1m]",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-pro[1m]",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  }
}
```

### 4.3 测试 DeepSeek 是否生效

```powershell
claude -p "用一句中文回复：Claude Code 已接入 DeepSeek。" --max-turns 1
```

如果出现 401 / 403，多半是 API Key 错误、DeepSeek API 余额不足或环境变量没有生效。

------

## 五、Windows 常用 CLI 命令速查

### 5.1 PowerShell 外部命令

| 命令 | 用途 |
|---|---|
| `claude` | 启动交互式会话 |
| `claude "解释这个项目"` | 带初始提示启动 |
| `claude -p "解释这个函数"` | 非交互模式，输出后退出 |
| `Get-Content .\logs.txt | claude -p "分析错误"` | 通过管道分析文件内容 |
| `claude -c` | 继续当前目录最近会话 |
| `claude -r "<session>"` | 恢复指定会话 |
| `claude update` | 更新 Claude Code |
| `claude install stable` | 安装/重装 stable 原生版本 |
| `claude doctor` | 诊断安装和配置 |
| `claude auth status` | 查看登录状态 |
| `claude mcp list` | 查看 MCP 服务器 |
| `claude plugin list` | 查看插件 |
| `claude --version` | 查看版本 |

官方还支持 `claude agents` 管理后台/并行会话、`claude project purge` 清理项目本地状态、`claude ultrareview` 做非交互深度审查等。([Claude Code](https://code.claude.com/docs/en/cli-reference))

### 5.2 常用参数

| 参数 | 用途 |
|---|---|
| `--model <model>` | 当前会话指定模型 |
| `--effort low/medium/high/xhigh/max` | 设置推理强度 |
| `--permission-mode plan` | 进入计划模式 |
| `--allowedTools "Read,Edit,Bash"` | 自动批准指定工具 |
| `--disallowedTools "Bash(Remove-Item *)"` | 禁止危险命令 |
| `--add-dir ..\lib` | 额外授权目录 |
| `--output-format json` | 非交互模式输出 JSON |
| `--max-turns 3` | 限制 agentic 轮数 |
| `--max-budget-usd 5.00` | 限制 API 花费 |
| `--worktree feature-x` | 在隔离 git worktree 中工作 |
| `--bare` | 跳过插件、MCP、CLAUDE.md 等自动发现，加速脚本调用 |

说明：Claude Code 文档里的 `Bash` 是命令执行工具名，不代表 Windows 必须安装 Bash。官方说明 `--bare` 适合 CI 和脚本，因为它会跳过 hooks、skills、plugins、MCP servers、auto memory 和 CLAUDE.md，启动更快且更可复现。([Claude Code](https://code.claude.com/docs/en/headless))

------

## 六、交互式常用 `/` 命令

在 `claude` 会话里输入 `/` 可以查看命令列表。命令必须出现在消息开头，后面的文字作为参数传入。([Claude Code](https://code.claude.com/docs/en/commands))

### 6.1 项目初始化

```text
/init
```

生成项目级 `CLAUDE.md`。

```text
/memory
```

编辑项目记忆和规则。

建议在 `CLAUDE.md` 写清楚：

```markdown
# 项目规则

- 包管理器：pnpm
- 启动：pnpm dev
- 测试：pnpm test
- 类型检查：pnpm typecheck
- 代码风格：提交前必须运行 pnpm lint
- 不要修改 migrations/ 下的历史迁移文件
```

### 6.2 开发与修改

```text
/plan 重构登录模块，先给方案不要改代码
```

进入计划模式，适合大改动。

```text
/diff
```

查看 Claude 改了哪些文件。

```text
/rewind
```

回退到之前的检查点。

```text
/compact
```

压缩上下文，长会话必备。

```text
/clear
```

开始新会话，保留项目记忆。

### 6.3 审查与安全

```text
/code-review
```

审查当前 diff。

```text
/code-review --fix
```

审查并尝试自动修复。

```text
/security-review
```

做安全审查。

```text
/simplify
```

做简化和清理类审查。

官方命令文档说明 `/code-review` 可按 effort 级别审查当前 diff，并支持 `--fix`；`/security-review` 用于分析当前分支待提交变更的安全风险。([Claude Code](https://code.claude.com/docs/en/commands))

### 6.4 插件、MCP、权限

```text
/plugin
```

打开插件管理器。

```text
/mcp
```

管理 MCP 连接和 OAuth 登录。

```text
/permissions
```

管理工具权限规则。

```text
/hooks
```

查看自动化 hooks。

### 6.5 会话管理

```text
/resume
```

恢复历史会话。

```text
/branch
```

从当前点分叉一个会话。

```text
/background
```

把当前任务放到后台。

```text
/tasks
```

查看后台任务。

```text
/exit
```

退出 CLI。

------

## 七、推荐插件

Claude Code 插件可以扩展 skills、agents、hooks、MCP servers 和 LSP servers。官方 marketplace `claude-plugins-official` 会自动可用；安装官方插件的格式是：([Claude Code](https://code.claude.com/docs/en/discover-plugins))

```text
/plugin install <插件名>@claude-plugins-official
```

### 7.1 强烈推荐：代码智能 LSP 插件

按语言安装：

| 语言 | 插件 | 需要本地二进制 |
|---|---|---|
| TypeScript | `typescript-lsp` | `typescript-language-server` |
| Python | `pyright-lsp` | `pyright-langserver` |
| Go | `gopls-lsp` | `gopls` |
| Rust | `rust-analyzer-lsp` | `rust-analyzer` |
| C/C++ | `clangd-lsp` | `clangd` |
| C# | `csharp-lsp` | `csharp-ls` |
| Java | `jdtls-lsp` | `jdtls` |
| PHP | `php-lsp` | `intelephense` |

这些插件让 Claude 获得 LSP 能力：跳转定义、查找引用、看到类型错误、编辑后自动诊断等。([Claude Code](https://code.claude.com/docs/en/discover-plugins))

示例：

```text
/plugin install typescript-lsp@claude-plugins-official
/plugin install pyright-lsp@claude-plugins-official
/plugin install gopls-lsp@claude-plugins-official
```

### 7.2 GitHub / GitLab

```text
/plugin install github@claude-plugins-official
/plugin install gitlab@claude-plugins-official
```

适合让 Claude 查看 issue、PR、提交、代码审查上下文。官方 marketplace 文档把 GitHub、GitLab 归在 source control 外部集成插件中。([Claude Code](https://code.claude.com/docs/en/discover-plugins))

### 7.3 项目管理与文档

```text
/plugin install linear@claude-plugins-official
/plugin install notion@claude-plugins-official
/plugin install atlassian@claude-plugins-official
```

适合从 Linear/Jira/Confluence/Notion 拉需求、同步文档。

## 八、Superpowers

Superpowers 是一个 Claude Code 技能框架，重点是给 Claude 加上更强的工程纪律：需求澄清、方案讨论、TDD、系统化调试、子代理开发、代码审查和技能编写。官方插件页说明它会引导 Claude 使用 `/brainstorming` 做需求和设计探索，用 `/execute-plan` 批量执行实现计划，并通过 code-reviewer agent 按计划、代码标准和架构原则审查实现。([Superpowers – Claude Plugin](https://claude.com/plugins/superpowers))

### 8.1 安装

在 Claude Code 会话里执行：

```text
/plugin install superpowers@claude-plugins-official
/reload-plugins
```

如果提示找不到插件，先刷新官方 marketplace：

```text
/plugin marketplace update claude-plugins-official
/plugin install superpowers@claude-plugins-official
/reload-plugins
```

也可以打开插件管理器安装：

```text
/plugin
```

然后进入 Discover，搜索 `superpowers` 并安装。

### 8.2 常用方式

```text
/brainstorming
```

适合在写代码前梳理需求、边界、方案和取舍。建议在新功能、大改动、UI 设计、架构调整前先用。

```text
/execute-plan
```

适合在已有实现计划后分批执行，并在关键节点进行审查。

```text
请使用 Superpowers 的系统化调试流程，分析这个 bug 的根因，不要直接猜测修复。
```

适合复杂 bug。Superpowers 的调试流程强调先做根因调查、模式分析和假设验证，再修改代码。

```text
请用 TDD 流程实现这个功能：先写失败测试，再做最小实现，然后重构。
```

适合核心业务逻辑、容易回归的功能、需要长期维护的模块。

### 8.3 推荐使用场景

- 新功能：先 `/brainstorming`，确认需求和边界，再让 Claude 写实现计划。
- 大重构：让 Claude 先列风险、影响面、测试策略，再执行。
- 修疑难 bug：要求使用系统化调试，避免 Claude 直接“猜修”。
- 团队协作：把 Superpowers 和 `/code-review`、`/security-review`、GitHub 插件一起用。

### 8.4 注意事项

Superpowers 会让流程更稳，但也会让 Claude 更“啰嗦”和更重视计划、测试、审查。小改动、一次性脚本、非常明确的微调任务，可以不用强制触发它。

## 九、CodeGraph

CodeGraph 严格来说更像 **MCP 代码图谱工具**，不是普通 Claude Code 插件。它会在本地把代码库解析成预索引的知识图谱，存到 SQLite，并通过本地 MCP server 给 Claude Code 查询。官方页面说明它基于 tree-sitter 解析代码，支持函数、类、调用链、导入关系等结构化查询；代码保留在本机，不需要 API Key，也不会上传到云端。([CodeGraph](https://codegraph.codes/))

### 9.1 适合解决什么问题

Claude Code 默认理解大型项目时，常会反复 `grep`、`glob`、`Read` 很多文件。CodeGraph 的作用是让 Claude 直接查“谁调用了这个函数”“改这个类影响哪些地方”“某个模块的调用链是什么”，减少盲扫代码和上下文浪费。CodeGraph 官方页面列出了 `codegraph_search`、`codegraph_context`、`codegraph_callers`、`codegraph_callees`、`codegraph_impact`、`codegraph_status` 等 MCP 工具。([CodeGraph](https://codegraph.codes/))

### 9.2 安装前准备

Windows 上建议先确认已有 Node.js 18+ 和 npm/npx：

```powershell
node --version
npm --version
npx --version
```

如果没有 Node.js，可以先安装：

```powershell
winget install OpenJS.NodeJS.LTS
```

安装后重新打开 PowerShell。

### 9.3 交互式安装

在项目目录中打开 PowerShell：

```powershell
cd C:\path\to\your-project
npx @colbymchenry/codegraph
```

安装器会自动检测 Claude Code、Cursor、Codex CLI 等 agent，询问要配置哪些工具，并写入 MCP server 配置和相关说明文件。CodeGraph 安装文档说明，它会提示是否把 `codegraph` 安装到 PATH、配置应用到全部项目还是当前项目，并在选择 Claude Code 时设置相应权限。([codegraph Installation](https://colbymchenry.github.io/codegraph/getting-started/installation/))

### 9.4 初始化项目索引

安装后在每个项目里执行：

```powershell
cd C:\path\to\your-project
codegraph init -i
```

这会为当前项目构建知识图谱索引，并配置项目本地的 agent 入口。CodeGraph 文档说明，完成一次全局安装后，在每个项目执行 `codegraph init -i` 即可让该项目可被查询。([codegraph Installation](https://colbymchenry.github.io/codegraph/getting-started/installation/))

### 9.5 验证 Claude Code 是否加载 CodeGraph

重启 Claude Code：

```powershell
claude
```

然后执行：

```text
/mcp
```

确认列表里有 `codegraph` 或相关 CodeGraph MCP server。

也可以问：

```text
请使用 CodeGraph 分析这个项目的主要模块、入口文件和调用关系。
```

```text
请使用 CodeGraph 查找 AuthService 的调用者、被调用函数，以及修改它的影响范围。
```

```text
请使用 CodeGraph 找出当前项目中可能未使用的函数，并按风险排序。
```

### 9.6 非交互安装 / CI 场景

如果已经安装过 `codegraph` 命令，可以用非交互方式：

```powershell
codegraph install --target=claude --location=global --yes
```

或只对当前项目生效：

```powershell
codegraph install --target=claude --location=local --yes
```

初始化：

```powershell
codegraph init -i
```

### 9.7 更新与卸载

更新通常重新运行安装器即可：

```powershell
npx @colbymchenry/codegraph
```

卸载：

```powershell
codegraph uninstall
```

只移除当前项目索引：

```powershell
codegraph uninit
```

CodeGraph 文档说明，`codegraph uninstall` 会移除它写入的 agent 配置、说明和权限；项目里的 `.codegraph/` 索引目录会保留，需要用 `codegraph uninit` 或手动删除。([codegraph Installation](https://colbymchenry.github.io/codegraph/getting-started/installation/))



### 9.8 安全与工作流

```text
/plugin install security-guidance@claude-plugins-official
/plugin install commit-commands@claude-plugins-official
/plugin install pr-review-toolkit@claude-plugins-official
```

`security-guidance` 会在 Claude 修改代码时检查常见漏洞；`commit-commands` 提供提交、push、PR 创建工作流；`pr-review-toolkit` 提供 PR 审查 agents。([Claude Code](https://code.claude.com/docs/en/discover-plugins))

### 9.9 社区插件市场

添加社区 marketplace：

```text
/plugin marketplace add anthropics/claude-plugins-community
```

安装格式：

```text
/plugin install <plugin-name>@claude-community
```

官方说明社区 marketplace 里的第三方插件通过了 Anthropic 的自动验证和安全筛查，并且 catalog 中会 pin 到具体 commit SHA。([Claude Code](https://code.claude.com/docs/en/discover-plugins))

### 9.10 插件管理命令

```text
/plugin
/plugin list
/plugin install plugin-name@marketplace-name
/plugin disable plugin-name@marketplace-name
/plugin enable plugin-name@marketplace-name
/plugin uninstall plugin-name@marketplace-name
/reload-plugins
/plugin marketplace list
/plugin marketplace update marketplace-name
/plugin marketplace remove marketplace-name
```

注意：插件和 marketplace 是高信任组件，可能在你的机器上以当前用户权限执行代码；只安装可信来源。官方安全说明也强调了这一点。([Claude Code](https://code.claude.com/docs/en/discover-plugins))

------

## 十、MCP 使用手册（Windows）

MCP，即 Model Context Protocol，可以让 Claude Code 使用内置工具以外的能力，比如浏览器、数据库、issue tracker、监控系统等。官方文档说明 MCP server 可以运行在本机，也可以是托管服务。([Claude Code](https://code.claude.com/docs/en/mcp-quickstart))

### 10.1 添加官方文档 MCP 示例

在普通 PowerShell 执行，不是在 Claude 会话里：

```powershell
claude mcp add --transport http claude-code-docs https://code.claude.com/docs/mcp
claude mcp list
```

进入 Claude：

```powershell
claude
```

然后问：

```text
Use the claude-code-docs server to look up what MCP_TIMEOUT does
```

### 10.2 添加 Playwright MCP

适合前端自动化测试、打开网页、点击按钮、检查页面。

```powershell
claude mcp add playwright -- npx -y @playwright/mcp@latest
claude mcp list
```

使用：

```text
Use playwright to open http://localhost:3000 and check the login page
```

官方 MCP 文档把 Playwright 作为本地 stdio MCP server 示例，说明它可以驱动浏览器导航、点击、读取页面。([Claude Code](https://code.claude.com/docs/en/mcp-quickstart))

### 10.3 添加 Sentry MCP

```powershell
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
claude mcp list
```

进入 Claude 后：

```text
/mcp
```

选择 `sentry`，完成浏览器 OAuth 登录。

### 10.4 MCP 配置文件

项目共享 MCP 配置可以写在项目根目录 `.mcp.json`：

```json
{
  "mcpServers": {
    "claude-code-docs": {
      "type": "http",
      "url": "https://code.claude.com/docs/mcp"
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

官方说明 MCP 有 local、project、user 三种 scope：local 和 user 存在 `%USERPROFILE%\.claude.json`，project 存在项目根目录 `.mcp.json`。([Claude Code](https://code.claude.com/docs/en/mcp-quickstart))

------

## 十一、自定义技能 / 命令（Windows PowerShell）

Claude Code 的 skills 可以把重复流程变成 `/命令`。官方说明：创建 `SKILL.md` 后，Claude 会在相关时自动使用，或者你可以直接用 `/skill-name` 调用；旧的 `.claude/commands/*.md` 也仍然可用。([Claude Code](https://code.claude.com/docs/en/skills))

### 11.1 创建一个代码审查技能

```powershell
New-Item -ItemType Directory -Force .\.claude\skills\api-review | Out-Null
@'
---
description: Review API changes for compatibility, security, and error handling
---

Review the selected or changed API code for:

1. Breaking changes
2. Auth and permission issues
3. Input validation
4. Error handling
5. Backward compatibility
6. Test coverage

Be concise and propose concrete patches when possible.
'@ | Set-Content -Encoding UTF8 .\.claude\skills\api-review\SKILL.md
```

在 Claude 会话里：

```text
/reload-skills
/api-review
```

### 11.2 创建部署检查命令

```powershell
New-Item -ItemType Directory -Force .\.claude\skills\deploy-check | Out-Null
@'
---
description: Check whether this branch is safe to deploy
---

Check the current branch before deployment:

- Inspect git diff
- Run tests if available
- Run lint/typecheck if available
- Check env/config changes
- Identify database migration risk
- Summarize go/no-go decision
'@ | Set-Content -Encoding UTF8 .\.claude\skills\deploy-check\SKILL.md
```

使用：

```text
/deploy-check
```

------

## 十二、Hooks 自动化

Hooks 可以在 Claude Code 生命周期的特定节点自动执行命令，比如：编辑后格式化、任务结束后通知、阻止修改敏感文件、审计配置变更等。官方说明 hooks 是确定性的控制机制，适合强制执行项目规则。([Claude Code](https://code.claude.com/docs/en/hooks-guide))

Windows 用户级配置文件通常位于：

```text
%USERPROFILE%\.claude\settings.json
```

如果目录不存在，可先创建：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude" | Out-Null
```

示例：任务需要注意时播放提示音。

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -Command \"[console]::beep(800,200)\""
          }
        ]
      }
    ]
  }
}
```

------

## 十三、推荐工作流（Windows）

### 13.1 第一次进入项目

```powershell
cd C:\path\to\your-project
claude
```

在 Claude 里执行：

```text
/init
```

然后让它补充项目规则：

```text
/memory
```

再问：

```text
请阅读项目结构，告诉我启动、测试、构建、发布分别怎么做，不要改代码。
```

### 13.2 开发一个功能

```text
/plan 实现用户头像上传，先给技术方案和需要修改的文件
```

确认方案后：

```text
按方案实现。每完成一个阶段先运行相关测试。
```

完成后：

```text
/diff
/code-review --fix
/security-review
```

### 13.3 修 bug

```powershell
claude "分析这个错误并定位根因：<粘贴报错>"
```

或者：

```powershell
Get-Content .\error.log | claude -p "分析这个日志，给出根因和修复建议"
```

### 13.4 非交互脚本调用

```powershell
claude -p "总结 src/auth 下的认证流程" --output-format json
```

CI 中建议：

```powershell
claude --bare -p "Review this diff for obvious bugs" --allowedTools "Read,Bash"
```

`-p` 是官方推荐的非交互模式，适合脚本和 CI；它支持 `--continue`、`--allowedTools`、`--output-format` 等选项。([Claude Code](https://code.claude.com/docs/en/headless))

------

## 十四、权限与安全建议

1. **不要一上来开启完全放权。** 大改动先用 `/plan`，确认方案后再允许编辑。
2. **谨慎使用 `--allowedTools`。** 可以允许 `Read`、`Edit`、安全的命令读取操作；不要随便允许删除文件、部署命令、生产数据库命令。
3. **把危险文件写进规则。** 比如 `.env.production`、`migrations/`、`terraform/`、`billing/`。
4. **插件只装可信来源。** 插件、MCP server、hooks 都可能执行本机命令。
5. **DeepSeek Key 不要提交到仓库。** 不要写进项目内的 `.claude/settings.json` 或 `.mcp.json` 后 commit。
6. **团队共享配置放 project scope，个人密钥放 user/local scope。** 官方配置 scope 包括 user、project、local、managed；user 适合个人偏好和密钥，project 适合团队共享规则，local 适合本机私有覆盖。([Claude Code](https://code.claude.com/docs/en/settings))
