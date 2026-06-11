# Claude Code CLI

## 简介

`Claude Code CLI`（以下简称 `Claude`） 是 Anthropic 官方的 **终端里的 AI 编程助手**。你在项目目录里运行 `claude`，就可以用自然语言让它读代码、解释项目、修改文件、写测试、修 bug、重构、生成提交信息、处理 Git 工作流等。官方把它定位为“agentic coding tool”，也就是能理解代码库并执行多步开发任务的编码代理。([GitHub](https://github.com/anthropics/claude-code "GitHub - anthropics/claude-code: Claude Code is an agentic coding tool that lives in your terminal, understands your codebase, and helps you code faster by executing routine tasks, explaining complex code, and handling git workflows - all through natural language commands. · GitHub"))

**它适合做什么：**

- 快速理解陌生项目：`what does this project do?`
    
- 找入口、解释目录结构、分析技术栈
    
- 修改代码：`fix the login bug...`
    
- 写测试、补文档、重构模块
    
- 辅助 Git：查看改动、写 commit message、解决冲突
    
- 一次性任务：`claude "fix the build error"` 或 `claude -p "explain this function"`([Claude Code](https://code.claude.com/docs/en/quickstart "Quickstart - Claude Code Docs"))

## 安装使用

### 安装

**macOS, Linux, WSL:**

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell:**

```powershell
irm https://claude.ai/install.ps1 | iex
```

**Windows CMD:**

```shell
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del i
```

**WinGet**

```
winget install Anthropic.ClaudeCode
```

推荐使用 `WinGet` 进行安装，否则对网络环境要求较高（DDDD）。

官方推荐在 Windows 上搭配 `Git for Windows` 的 bash 环境使用，因为 `Claude` 可使用各种 bash 工具，若未安装  `Git for Windows` 则将 fallback 使用 `PowerShell`。

> 注意：只需安装 `Git for Windows` 并添加到环境变量即可，不是必须在 git bash 中使用

`PowerShell` 分为 `Windows PowerShell` 和 `PowerShell 7`，微软官方推荐使用 `Windows Terminal` + `PowerShell 7` 以获得更现代化的终端使用体验。事实上，在 Windows 平台， `Windows Terminal` + `PowerShell 7` 是公认的最强组合。

`Windows Terminal` 和 `PowerShell 7` 均可在微软应用商店下载安装后使用，`Windows Terminal` 将自动识别到 `PowerShell 7`，只需在设置中将 `PwoerShell 7` 设置为默认配置，每次打开终端即可自动使用 `PowerShell 7`。

> 注意：微软商店安装版本会有部分功能限制，且无法被其他如 VS Code 使用

若是想在如 `VS Code`， `JetBrains`， `Qt Creator` 等其他软件中使用 `PowerShell 7`，则建议手动安装（[PowerShell - Github releases](https://github.com/PowerShell/PowerShell/releases/)），以获得确切的安装位置。安装时请勿修改安装位置，否则 `Windows Terminal` 无法自动识别到，需手动配置（较繁琐）。

### 配置以使用 DeepSeek

参考文档 [DeepSeek API 文档 - 接入 Agent 工具/Claude Code](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)

需要一个 DeepSeek API Key，在 [DeepSeek Platform](https://platform.deepseek.com/api_keys) 获取，注意妥善保存。

配置方式有以下两种：

1. 遵循官方示例，通过用户级环境变量持久化配置，**注意替换 API Key**：

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

2. 写入 `Claude` 的配置文件：

打开终端，使用 `notepad` 或 `VS Code` 或其他编辑器打开 `~/.claude/settings.json`

> 也可找到文件手动打开

```shell
notepad ~/.claude/settings.json
# 或者
code ~/.claude/settings.json
```

修改内容为 `env` 块为以下内容：

```json
{
"env": {
	"ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "<你的 DeepSeek API Key>",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash"
    "CLAUDE_CODE_EFFORT_LEVEL": "ultracode",
    "CLAUDE_CODE_NEW_INIT": "1"
  }
}
```

`DeepSeek` 提供 `deepseek-v4-pro` 和 `deepseek-v4-flash` 两种模型，均提供 1M 长度的上下文能力，这里的 `[1m]` 后缀用于让 `Claude` 知道该模型支持 1M 长度的上下文。 

可自由设置模型，例如全设置为 `deepseek-v4-pro[1m]` 以始终使用 `deepseek-v4-pro` 获得最强性能。
### 开始使用

1. 启动 `Claude`：打开项目目录，路径栏输入 `claude` 即可自动打开，若是配置了 `PowerSehll 7` 则将自动使用
2. 若是第一次打开该目录， `Claude` 将提示你是否相信该目录，选择相信即可
## 使用方法

### 推荐组合

若是偏向终端中使用，推荐上方提到的 `Windows Terminal` + `PowerShell 7` 组合。

若是偏向在 `VS Code` 中使用，则可在 `VS Code` 中安装插件 `# Claude Code for VS Code`，然后在右边栏中点击 `CLAUDE CODE` 字样切换到 `Claude` 进行使用。

若是偏向使用桌面应用，则需下载 `Claude Code Desktop`，并搭配 `CC Switch` 进行使用。

个人推荐在 `VS Code` 中使用，界面比终端更加方便的同时，功能并未阉割，且可同时使用 `VS Code` 的其他功能。

### 基础使用

`Claude Code` 提供多种工作模式，常用模式可通过 `Shift + Tab` 循环切换。不同界面中的显示名称略有差异。

| 模式            | 终端显示              | VS Code 显示           | 行为说明                                                           | 适用场景                        |
| ------------- | ----------------- | -------------------- | -------------------------------------------------------------- | --------------------------- |
| `default`     | 无显示               | `Ask before edits`   | 默认模式。<br>读取文件等只读行为通常无需确认；修改文件、执行可能产生影响的命令时需要用户批准               | 初次使用、敏感仓库、希望逐步审查每一步操作       |
| `plan`        | `plan mode on`    | `Plan mode`          | 计划模式。<br>Claude 会先阅读仓库、理解需求、制定实现计划；用户确认计划后再进入实现阶段              | 大改动、重构、复杂 bug 修复、需要先评审方案的任务 |
| `acceptEdits` | `accept edits on` | `Edit automatically` | 自动接受修改模式。<br>Claude 可以自动创建或修改工作目录内的文件，用户之后通过编辑器或 `git diff` 审查 | 频繁小改动、测试补全、文档修改、低风险重构       |
| `auto`        | `auto mode on`    | `Auto mode`          | 自动模式。<br>Claude 会尽量减少常规权限确认，但危险、不可逆、越界或不明确的操作仍会被拦截或要求确认        | 长任务、多步骤实现、希望减少确认打断的场景       |

> 注意：`auto` 模式并非所有环境都会出现，是否可用取决于 Claude Code 版本、账号权限、模型支持和组织配置。安全要求较高的项目不建议一开始就使用 `auto`。

常见启动方式：

```bash
# 在当前项目中启动交互式会话
claude

# 启动时指定权限模式
claude --permission-mode plan
claude --permission-mode acceptEdits

# 直接让 Claude 执行一次性任务
claude "explain this repository"
claude "fix the failing tests"
```

Claude Code 具有大量内建命令，用于在会话中快速切换模式、管理上下文、调整模型、查看变更、管理权限、恢复会话和排查问题。命令需要写在消息开头，例如 `/help`、`/clear`、`/plan fix bug`；如果命令后面继续输入文字，这些文字会作为命令参数传入。

常用命令总览：

| 命令                                    | 用途                      | 推荐场景                         |
| ------------------------------------- | ----------------------- | ---------------------------- |
| `/help`                               | 查看帮助和当前可用命令             | 不确定某个命令是否存在或如何使用时            |
| `/status`                             | 查看版本、模型、账号和连接状态         | 排查环境、模型或登录问题                 |
| `/config`<br>`/settings`              | 打开设置界面                  | 调整主题、模型、输出风格等偏好              |
| `/model`                              | 切换模型                    | 需要在速度、成本、能力之间切换时             |
| `/effort`                             | 调整推理投入程度                | 简单任务降 effort，复杂任务升 effort    |
| `/init`                               | 初始化/更新 `CLAUDE.md`      | 项目初次使用 `Claude` 或进行更新        |
| `/plan`                               | 进入计划模式                  | 复杂修改、多文件变更、重构前先规划            |
| `/clear`<br>`/reset`<br>`/new`        | 开启新会话并清空当前上下文           | 开始完全不同的新任务                   |
| `/compact`                            | 压缩当前长会话上下文              | 会话太长但仍想继续当前任务（不推荐）           |
| `/context`                            | 查看上下文使用情况               | 判断是否需要 `/compact` 或清理无关内容    |
| `/btw`                                | 提出不会污染主上下文的临时问题         | 中途问一个旁支问题                    |
| `/memory`                             | 编辑或查看 `CLAUDE.md` 记忆文件  | 维护项目背景、命令和规范                 |
| `/permissions`<br>`/allowed-tools`    | 管理工具权限规则                | 减少或收紧 Claude 执行命令、读写文件时的确认   |
| `/mcp`                                | 管理 MCP server 连接        | 启用、禁用或重连 MCP 服务              |
| `/plugin`                             | 管理插件                    | 安装、启用、禁用插件                   |
| `/agents`                             | 管理 subagent 配置          | 需要 Claude 委派子任务或使用专用 agent   |
| `/hooks`                              | 查看 hooks 配置             | 排查自动检查、自动格式化、安全扫描等 hook      |
| `/ide`                                | 管理 IDE 集成状态             | 确认 VS Code、JetBrains 等集成是否正常 |
| `/diff`                               | 查看当前未提交改动               | 提交前 review 当前变更              |
| `/code-review`                        | 检查当前 diff 的正确性、复用性和简化空间 | 提交前进行代码审查                    |
| `/review`                             | 审查 Pull Request         | 本地审查 PR                      |
| `/security-review`                    | 检查当前分支待提交改动的安全风险        | 涉及鉴权、输入处理、CI/CD、后端接口时        |
| `/simplify`                           | 检查并简化已改代码               | 希望减少重复、降低复杂度、改进抽象层次          |
| `/rewind`<br>`/checkpoint`<br>`/undo` | 回退代码或会话到之前的检查点          | Claude 改偏了、需要撤回某段修改时         |
| `/resume`<br>`/continue`              | 恢复之前的会话                 | 继续早前中断的任务                    |
| `/branch`                             | 从当前会话分叉出一条新方向           | 想尝试另一种方案但保留当前进度              |
| `/copy`                               | 复制最近一次回复                | 复制回答、代码块或保存到文件               |
| `/export`                             | 导出当前会话                  | 归档分析过程或交接给别人                 |
| `/doctor`                             | 诊断 Claude Code 安装和设置    | Claude Code 异常、命令不可用、配置有问题时  |
| `/debug`                              | 开启并分析调试日志               | 排查 Claude Code 运行时问题         |
| `/feedback`<br>`/bug`                 | 提交反馈或 bug 报告            | 遇到工具问题需要上报时                  |
| `/usage`<br>`/cost`<br>`/stats`       | 查看用量、费用或限制状态            | 关注 token、额度或使用统计时            |
| `/release-notes`                      | 查看版本更新日志                | 升级后查看新增命令或行为变化               |
| `/exit`<br>`/quit`                    | 退出 Claude Code          | 结束当前 CLI 会话                  |

推荐工作流：

```text
/init
  # Claude：阅读仓库代码，生成或改进 CLAUDE.md
  # 作用：帮助 Claude 后续理解项目结构、常用命令、代码规范和注意事项
  ↓
人工审查 CLAUDE.md
  # 人：补全项目背景、业务规则、特殊注意事项、禁止事项、常用命令
  # 重点检查：Claude 是否误解项目结构、测试命令、构建方式和模块职责
  ↓
/memory
  # Claude：查看和编辑当前会话加载的 CLAUDE.md / CLAUDE.local.md / rules
  # 人：确认当前会话实际加载了正确的项目记忆
  ↓
/permissions
  # Claude：管理工具权限规则
  # 人：配置哪些 Bash、文件读写、MCP 工具可自动执行，哪些需要确认，哪些禁止执行

  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

开始复杂任务前
  ↓
/effort ultracode
  # Claude：启用面向大型任务的高推理工作流模式
  # 适合：复杂实现、系统性重构、多文件改动、全面审计
  ↓
/plan 实现用户登录重构（可使用 /brainstorm 替代，推荐）
  # Claude：先分析需求、拆解步骤、识别风险，而不是直接修改代码
  ↓
人工审查 plan
  # 人：确认目标、范围、风险点和执行顺序
  # 必要时要求 Claude 修改计划，避免一开始就改偏

  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

执行修改
  ↓
Claude 按 plan 修改代码
  # 人：关注 Claude 是否超出任务范围、是否删除不该删除的逻辑
  ↓
/diff（可在 VS Code 中插件，更加直观方便）
  # Claude：展示当前未提交改动和本轮修改内容
  # 人：人工 review 关键文件、核心逻辑和意外变更
  ↓
/code-review
  # Claude：审查当前 diff
  # 重点：正确性 bug、复用性、简化空间、效率问题
  ↓
/security-review
  # Claude：审查当前分支待提交改动中的安全风险
  # 重点：注入、鉴权、敏感信息泄露、权限问题
  ↓
人工最终确认
  # 人：运行测试、检查结果、确认是否可以提交

  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

会话变长后继续（不推荐，两个任务没有明显关联时，推荐直接 /clear 开启新对话）
  ↓
/context
  # Claude：查看当前上下文占用情况
  # 人：判断是否因为文件、历史对话或工具输出导致上下文膨胀
  ↓
/compact
  # Claude：压缩当前会话上下文，用摘要替代长历史
  # 注意：会丢失大量细节，复杂任务中不建议使用
  ↓
人工补充关键上下文
  # 人：重新说明当前任务目标、已完成内容、未完成内容、关键约束
  # 避免 compact 后 Claude 遗漏重要细节

  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

出问题时回退
  ↓
发现 Claude 改偏或破坏代码
  ↓
/rewind
  # Claude：回退到之前的检查点
  # 作用：撤销 Claude 改偏的代码，或回到较早的对话状态
  ↓
人工重新明确边界
  # 人：说明哪些文件不能改、哪些方案不可接受、下一步只允许做什么

  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

开始新任务
  ↓
确认当前任务已完成或不再需要上下文
  ↓
/clear
  # Claude：清空当前会话上下文，开始一个新任务
  # 注意：项目级 CLAUDE.md 记忆仍会保留
  ↓
重新描述新任务目标
  # 人：提供新的任务背景、范围、验收标准和限制条件
```

### 插件系统

`Claude Code` 提供 `/plugin` 命令打开插件系统。插件可以为 Claude Code 增加额外能力，例如：

- `skills`：可复用的专门任务能力
    
- `agents`：专门处理某类任务的子代理
    
- `hooks`：在特定事件发生时自动执行逻辑
    
- `MCP servers`：接入外部工具、服务或数据源
    
- `LSP servers`：提供代码跳转、引用查找、类型诊断等代码智能能力

在 Claude Code 中输入：

```text
/plugin
```

即可打开插件管理界面。

插件使用建议：

- 插件安装后运行 `/reload-plugins` 重新加载
    
- 如果插件报错，先查看 `/plugin` 的 `Errors` 页面
    
- 团队项目中可以将插件配置固定到项目范围，保证成员使用一致的开发体验
#### 推荐安装

##### `clangd-lsp`
  
`clangd-lsp` 为 Claude Code 提供 C/C++ 语义级代码理解能力，使 Claude 可以借助 `clangd` 获取类型信息、跳转定义、查找引用、查看诊断和理解函数调用关系，适合大型 C/C++ 项目、复杂头文件依赖、模板、宏定义较多的代码库。

本机需事先安装 `clangd` 否则无法使用，使用 `winget install LLVM.LLVM` 安装

建议同时生成 `compile_commands.json`，否则 `clangd` 可能无法正确理解编译参数、头文件路径和宏定义。

```bash
# CMake 项目生成 compile_commands.json
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# macOS / Linux：可选，链接到项目根目录
ln -s build/compile_commands.json compile_commands.json

# Windows PowerShell：可选，复制到项目根目录
Copy-Item build/compile_commands.json compile_commands.json
```

最常用命令和使用方式：

```bash
# Windows 安装 clangd
winget install LLVM.LLVM

# 安装 Claude Code 插件
/plugin install clangd-lsp@claude-plugins-official
```

常用提问方式：

```
检查当前 C++ 文件的诊断并修复
查找这个 struct 的所有引用
跳转并解释这个函数的定义和调用链
分析这个头文件为什么无法被 clangd 正确识别
```

##### `superpowers`

`superpowers` 为 Claude Code 提供结构化软件开发工作流能力，适合需求澄清、方案设计、系统化调试、TDD、代码审查、分阶段执行复杂任务等场景。它的价值在于让 Claude 不直接“上手改代码”，而是先拆解问题、制定计划、验证假设，再执行实现。

最常用命令

```
/brainstorming
/execute-plan
```

常用提问方式：

```
使用 brainstorming 帮我梳理这个功能的实现方案
先不要写代码，先用 superpowers 帮我拆解需求和风险
使用系统化调试方法分析这个 bug 的根因
根据当前计划分阶段执行，并在关键节点进行 review
```

##### `claude-code-setup`

`claude-code-setup` 用于分析当前代码库，并推荐适合项目的 Claude Code 自动化配置，例如 hooks、skills、MCP servers、subagents 和 slash commands。它适合新项目接入 Claude Code 时使用，帮助快速判断哪些插件、自动化和项目级配置最值得优先启用。

常用提问方式：

```
help me set up Claude Code for this project
recommend automations for this project
what hooks should I use?
what MCP servers should I use for this codebase?
analyze this project and suggest the best Claude Code setup
```

##### `claude-md-management`

`claude-md-management` 用于维护 `CLAUDE.md` 项目记忆文件，帮助 Claude 审计文档质量、发现上下文缺口，并在开发会话结束后沉淀新的命令、代码约定、环境问题和踩坑经验。它适合长期维护项目，能减少 Claude 每次重新理解项目的成本。

常用命令：

```
/revise-claude-md
```

常用提问方式：

```
audit my CLAUDE.md files
check if my CLAUDE.md is up to date
review my CLAUDE.md and suggest improvements
把本次会话中发现的构建命令、测试命令和注意事项整理进 CLAUDE.md
```

建议写入 `CLAUDE.md` 的内容：

```
项目简介和核心模块
构建、运行、测试、格式化命令
目录结构和关键文件说明
代码风格、命名规范和提交规范
常见问题、环境限制和禁止事项
```

##### `commit-commands`

`commit-commands` 用于规范和自动化 Git 提交流程，帮助 Claude 根据当前改动生成清晰的 commit message，并支持提交、推送、创建 PR 等常见 Git 工作流。适合需要维护清晰 Git 历史、遵循 Conventional Commits 或自动生成 changelog 的项目。

常用命令：

```
/commit
/commit-push-pr
/clean_gone
```

常用提问方式：

```
检查当前 diff，并生成符合 Conventional Commits 的提交信息
帮我把当前改动拆成合理的 commit
提交前检查 staged changes 是否和 commit message 匹配
提交、推送并创建 PR
```

常见提交类型：

```
feat: 新功能
fix: 修复问题
refactor: 重构
docs: 文档修改
test: 测试相关
chore: 构建、依赖、工具配置等杂项
```

##### `context7`

`context7` 用于为 Claude 提供更新、版本相关的第三方库和框架文档上下文，减少 Claude 依赖过时记忆生成错误 API 的问题。适合查询框架、SDK、库函数、配置项和示例代码，尤其适合 Next.js、React、Supabase、Cloudflare、Prisma 等快速变化的技术栈。

最常用命令和使用方式：

```
# 使用 Context7 CLI 为 Claude Code 设置（需事先安装 nodejs ）
npx ctx7 setup --claude

# 查询库 ID
ctx7 library <library-name> <query>

# 查询指定库文档
ctx7 docs <library-id> <query>
```

常用提问方式：

```
Create a Next.js middleware that checks JWT cookies. use context7
Show me the Supabase auth API for email/password sign-up. use context7
How do I configure Prisma with PostgreSQL? use context7
Use library /vercel/next.js to explain App Router middleware
```

建议写入 `CLAUDE.md` 的规则：

```
Always use Context7 when I need library/API documentation, code generation, setup or configuration steps without me having to explicitly ask.
```

### MCP

`MCP` 全称是 `Model Context Protocol`，即模型上下文协议。它用于把 Claude Code 连接到外部工具、服务或数据源，让 Claude 不只是在本地读写代码，还可以调用更多外部能力。

常见用途包括：

1. 连接 GitHub / GitLab，读取 issue、PR、CI 信息
    
2. 连接 Jira / Linear，查询和更新任务
    
3. 连接 Slack / Discord，读取讨论上下文
    
4. 连接数据库，查询 schema 或运行受控查询
    
5. 连接浏览器自动化工具，例如 Playwright
    
6. 连接公司内部文档、接口平台、知识库或自研工具

可以把 MCP 理解为：  
**给 Claude Code 安装“外部工具接口”。**

#### MCP 的基本管理命令

MCP 服务器通常在 Claude 会话外配置，也就是在普通终端中运行：

```bash
# 查看已配置的 MCP server
claude mcp list

# 查看某个 MCP server 的配置
claude mcp get <name>

# 移除某个 MCP server
claude mcp remove <name>
```

在 Claude Code 会话内部，可以使用：

```text
/mcp
```

打开 MCP 管理界面，查看连接状态、进行认证或管理已有 MCP server。

#### 添加远程 HTTP MCP Server

示例：

```bash
claude mcp add --transport http claude-code-docs https://code.claude.com/docs/mcp
```

含义：

- `claude mcp add`：添加 MCP server
    
- `--transport http`：使用远程 HTTP 服务
    
- `claude-code-docs`：自定义 server 名称
    
- 最后的 URL：MCP server 地址
    

添加后检查状态：

```bash
claude mcp list
```

常见状态：

|状态|含义|
|---|---|
|`Connected`|已连接，可使用|
|`Needs authentication`|需要登录或授权|
|`Failed to connect`|连接失败|
|`Pending approval`|项目级 MCP server 等待用户批准|

#### 添加本地 MCP Server

本地 MCP server 通常通过 `stdio` 启动，也就是 Claude Code 在本机启动一个子进程作为工具服务。

示例：添加 Playwright MCP，用于浏览器自动化：

```bash
claude mcp add playwright -- npx -y @playwright/mcp@latest
```

这里 `--` 后面的内容是实际启动 MCP server 的命令。

使用场景：

```text
使用 playwright 打开本地页面 http://localhost:3000，检查登录按钮是否可点击
```

#### MCP 配置作用域

MCP server 可以配置在不同作用域中：

|作用域|配置位置|适用范围|是否适合提交到仓库|
|---|---|---|---|
|`local`|用户配置中当前项目条目|仅当前用户、当前项目|否|
|`user`|用户全局配置|当前用户的所有项目|否|
|`project`|项目根目录 `.mcp.json`|克隆该项目的团队成员|是，但需谨慎|

示例：添加到用户全局作用域：

```bash
claude mcp add --scope user --transport http docs https://example.com/mcp
```

示例：添加到项目作用域：

```bash
claude mcp add --scope project --transport http internal-docs https://example.com/mcp
```

项目作用域会生成或修改 `.mcp.json`，适合团队共享标准化工具配置。但注意不要把密钥、token、个人凭证直接提交进仓库。

#### 直接从 JSON 添加 MCP Server

如果某个 MCP server 提供了 JSON 配置，可以使用：

```bash
claude mcp add-json <name> '<json>'
```

示例：

```bash
claude mcp add-json weather-api '{"type":"http","url":"https://api.example.com/mcp"}'
```

本地 stdio 示例：

```bash
claude mcp add-json local-tool '{"type":"stdio","command":"/path/to/tool","args":["--mode","mcp"]}'
```

#### MCP 使用建议

1. 只添加可信来源的 MCP server
    
2. 优先使用官方或团队维护的 MCP server
    
3. 不要把个人 token、API key、cookie 写入可提交的 `.mcp.json`
    
4. 对能写入外部系统的 MCP 工具要保持谨慎，例如发消息、改 issue、操作数据库
    
5. 第一次使用新 MCP 工具时，观察 Claude 输出中的工具调用名称，确认它确实调用了预期 server
    
6. 不常用的 MCP server 建议移除，减少上下文占用和潜在风险

#### 推荐 MCP

##### `context7`

获取最新技术文档，保证模型使用最新文档（也可通过插件安装）。适合查询第三方库、框架、SDK、配置项和版本相关 API，减少 Claude 使用过时接口或编造不存在 API 的概率。常用于快速变化的技术栈。

安装，需 nodejs 环境：

```bash
npx ctx7 setup --claude
```

常用提问方式：

```
使用 context7 查询 Next.js middleware 的最新写法
根据 Supabase 最新文档实现邮箱密码登录，use context7
查询 Prisma 连接 PostgreSQL 的配置方式，use context7
使用 library /vercel/next.js 查询 App Router 相关文档
```

建议写入 `CLAUDE.md`：

```
当任务涉及第三方库、框架、SDK、API、配置项、安装步骤或示例代码时，优先使用 Context7 获取最新文档，不要仅依赖模型记忆。
```

##### `codegtaph`

为 Claude 提供代码图谱能力，帮助模型从“纯文本搜索”升级为“按符号、调用关系、依赖关系和影响范围理解代码”。适合大型仓库、跨模块调用链分析、重构影响评估、查找调用方/被调用方、定位业务流程入口，以及减少 Claude 反复 grep 和读取无关文件带来的 token 消耗。

安装：

```bash
# 安装本地 CodeGraph CLI
npm install -g @colbymchenry/codegraph
# 配置到 Claude Code / Cursor 等 Agent
codegraph install
```

常用命令：

```bash
# 在项目根目录初始化代码图谱索引
codegraph init -i

# 增量更新索引
codegraph sync
```

常用提问方式：

```text
使用 codegraph 分析登录请求从路由到数据库的完整调用链

查找 UserService.login 的所有调用方和影响范围

修改这个接口前，先用 codegraph 分析会影响哪些模块

用 codegraph 找出认证模块的核心入口、关键类型和调用关系
```

建议写入 `CLAUDE.md`：

```text
在分析大型代码库、跨文件调用链、重构影响范围、函数调用关系或模块边界时，优先使用 CodeGraph / codegraph MCP，不要只依赖 grep、find 或全文搜索。
```

## 推荐工作流

### 1. 理解项目

先让 Claude 阅读仓库，了解项目结构、主要模块、启动方式和测试方式，不要直接修改代码。

```text
请先阅读这个仓库，说明项目结构、主要模块、启动方式和测试方式。不要修改代码。
```

新项目建议先执行：

```text
/init
```

然后人工审查 `CLAUDE.md`，补充项目背景、业务规则、特殊注意事项和常用命令。

如果已安装 `claude-code-setup`，可以让 Claude 分析当前项目适合启用哪些配置、插件、MCP、hooks 和 subagents：

```text
help me set up Claude Code for this project
```

或：

```text
analyze this project and suggest the best Claude Code setup
```

### 2. 维护项目记忆

使用 `CLAUDE.md` 沉淀项目上下文，避免 Claude 每次都重新理解项目。

```text
/memory
```

如果已安装 `claude-md-management`，建议让它审查并改进 `CLAUDE.md`：

```text
audit my CLAUDE.md files
```

或：

```text
/revise-claude-md
```

重点补充：

```text
项目背景
目录结构
构建、运行、测试命令
代码规范
特殊注意事项
禁止事项
常见问题
```

### 3. 确认需求

复杂任务开始前，先让 Claude 复述需求，确认目标、范围、风险和不应修改的内容。

```text
请先复述你对需求的理解，说明修改范围、风险点和不应该改动的内容。不要修改代码。
```

如果需求还不清楚，可以使用 `superpowers` 的头脑风暴流程先澄清需求：

```text
/brainstorming
```

或直接说明：

```text
使用 brainstorming 帮我梳理这个需求，先不要写代码。
```

后续步骤将自动使用 `superpowers` 定义的流程，当然，`superpowers` 的能力也可单独使用。
### 4. 制定计划

切换到 `plan` 模式，让 Claude 先分析问题并制定执行步骤。

```text
/plan 分析登录模块的鉴权流程，找出 token 失效后无法刷新的原因，并给出修改计划。
```

如果任务较复杂，建议配合 `superpowers`，让 Claude 按结构化流程拆解：

```text
使用 superpowers 帮我制定实现计划，包含涉及文件、修改步骤、风险点和验证方式。
```

人工确认计划后再执行，避免一开始就改偏或过度重构。

### 5. 执行修改

按确认后的计划分阶段修改。高风险任务建议手动批准，低风险任务可使用 `acceptEdits` 或 `auto` 减少确认。

```text
按刚才确认的计划执行修改。每完成一个阶段后说明修改了哪些文件和原因。
```

如果已安装 `superpowers`，可以使用：

```text
/execute-plan
```

执行时建议要求：

```text
按计划分阶段执行，每阶段完成后停下来总结，不要一次性改完整个项目。
```

### 6. 审查改动

建议在 VS Code 中审查改动，在终端中审查较为麻烦，用法如下：

修改完成后先看 diff，确认没有无关改动或错误修改。

```text
/diff
```

也可以使用：

```bash
git diff
```

涉及复杂逻辑时建议运行：

```text
/code-review
```

涉及鉴权、权限、输入处理、CI/CD 或后端接口时建议运行：

```text
/security-review
```

### 7. 编译验证

通常 `Claude` 会自动尝试构建/编译，但可能在某些情况下无法进行，则需要手动进行，然后将构建信息贴给 `Claude`。

先运行构建或编译命令，确认项目能正常构建。

```text
运行项目的编译或构建命令，并根据失败信息继续修复。只修复和本次任务相关的问题。
```

也可以明确指定命令：

```text
运行 npm run build，如果失败，请分析错误原因并修复。
```

### 8. 测试验证

补充或运行相关测试，优先验证本次改动影响的模块，不要一开始就跑无关的大范围测试。

```text
只运行 auth 相关测试，不要运行全量测试。
```

如需补充测试：

```text
请为本次修改补充最小必要测试，覆盖正常场景和异常场景。
```

如果已安装 `superpowers`，可要求 Claude 使用 TDD 或系统化验证方式：

```text
使用 superpowers 的测试驱动思路，为本次修改补充测试并验证。
```

### 9. 总结提交

最后让 Claude 总结修改内容、影响范围、验证情况和风险点，并生成提交信息。

```text
总结本次修改内容、影响范围、测试情况，并生成一条合适的 commit message。
```

如使用 Conventional Commits（推荐，提高 commit 可读性，也方便 CI 流水线自动生成 CHANGELOG）：

```text
请根据当前 diff 生成符合 Conventional Commits 的 commit message。
```

任务结束后，建议使用 `claude-md-management` 将本次新增的重要经验沉淀到 `CLAUDE.md`：

```text
把本次任务中发现的构建命令、测试命令、踩坑点和项目约定整理进 CLAUDE.md。
```

### 推荐完整流程

```text
理解项目
  ↓
使用 claude-code-setup 检查项目配置建议
  ↓
初始化 / 审查 CLAUDE.md
  ↓
使用 claude-md-management 维护项目记忆
  ↓
确认需求
  ↓
使用 superpowers 梳理复杂需求
  ↓
制定计划
  ↓
执行修改
  ↓
审查改动
  ↓
编译验证
  ↓
测试验证
  ↓
总结提交
  ↓
沉淀经验到 CLAUDE.md
```