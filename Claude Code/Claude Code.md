# Claude Code

> 版本：v1.0
> 更新日期：2026-06-03
> 适用对象：独立开发者、工程团队、技术负责人、产品工程师、AI 编程工作流使用者
> 目标：让 Claude Code 更准确、更可控、更安全地完成真实开发任务，而不是只生成“看起来能跑”的代码。

---

## 一、一句话理解 Claude Code

Claude Code 不是普通聊天机器人，而是一个运行在开发环境里的 AI 编程代理。它可以读取代码库、理解项目结构、编辑文件、运行命令、执行测试、使用 Git、调用外部工具，并在你的监督下完成开发任务。

最佳使用方式不是“让它随便写代码”，而是：

```text
明确目标 → 让它先探索 → 让它制定计划 → 执行小步修改 → 自动验证 → 人类审查 → 提交 PR
```

---

## 二、核心原则

### 2.1 先探索，再计划，再编码

不要一上来就说：

```text
帮我实现支付功能。
```

更好的方式：

```text
先不要修改代码。请阅读当前项目中与支付、订单、用户账户相关的文件，回答：
1. 当前支付流程是怎样的？
2. 哪些文件最可能需要修改？
3. 有哪些风险点？
4. 你建议的实现计划是什么？
等我确认计划后再开始改代码。
```

适用于以下场景：

* 新功能涉及多个文件
* 你不熟悉当前代码库
* 需求还不够明确
* 修改可能影响核心业务逻辑
* 涉及数据库、鉴权、支付、权限、安全、部署等高风险区域

---

### 2.2 给 Claude 可验证的成功标准

Claude Code 最容易出错的情况是：任务没有客观完成标准。每次任务都应尽量提供验证方式。

低质量提示：

```text
修复这个 bug。
```

高质量提示：

```text
修复登录页在 session 过期后白屏的问题。
请先写一个能复现该问题的测试，再修复代码。
完成后运行相关测试，并把测试命令和结果贴出来。
不要通过隐藏错误、删除异常处理或跳过测试来让测试通过。
```

常见验证方式：

* 单元测试
* 集成测试
* 类型检查
* Lint
* 构建命令
* 截图对比
* 日志检查
* API 响应对比
* 手动验收清单

---

### 2.3 小步快跑，不要一次交给它整个系统

Claude Code 能做大任务，但不代表应该一次性做完。最佳实践是把任务拆成可验证的小块。

推荐拆分方式：

```text
第一步：只分析现状，不改代码。
第二步：只制定实现计划。
第三步：只修改后端接口。
第四步：补测试并运行。
第五步：修改前端调用。
第六步：整体回归并总结 diff。
```

这样可以降低上下文膨胀、误改文件、方向跑偏和难以 review 的风险。

---

## 三、安装与启动

### 3.1 安装方式

macOS、Linux、WSL：

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Windows PowerShell：

```powershell
irm https://claude.ai/install.ps1 | iex
```

Windows CMD：

```cmd
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

Homebrew：

```bash
brew install --cask claude-code
```

WinGet：

```powershell
winget install Anthropic.ClaudeCode
```

### 3.2 登录

进入项目目录后运行：

```bash
claude
```

首次运行会要求登录。之后可在会话中使用：

```text
/login
```

切换或重新认证账号。

### 3.3 接入 DeepSeek

通过终端打开 `Claude Code` 的设置文件

```bash
notepad %USERPROFILE%\.claude\settings.json
```

找到 `env` 块替换或填入以下内容：

```json
{
    "env": {
        "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
        "ANTHROPIC_AUTH_TOKEN": "<你的 DeepSeek API Key>",
        "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
        "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
        "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
        "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
        "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
        "CLAUDE_CODE_EFFORT_LEVEL": "max"
    }
}
```

若想获得最强的性能，可将所有模型均替换为 `deepseek-v4-pro[1m]`

其他实现方法可参考 [接入 Claude Code | DeepSeek API Docs](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/claude_code)

### 3.4 启动项目会话

```bash
cd /path/to/your/project
claude
```

启动后建议先问：

```text
请先阅读项目结构，告诉我：
1. 这个项目是做什么的？
2. 使用了哪些主要技术栈？
3. 主入口在哪里？
4. 测试、构建、启动命令分别是什么？
不要修改任何文件。
```

---

## 四、日常高频命令

| 命令                  | 用途                | 示例                                  |
| ------------------- | ----------------- | ----------------------------------- |
| `claude`            | 启动交互模式            | `claude`                            |
| `claude "task"`     | 运行一次性任务           | `claude "fix the build error"`      |
| `claude -p "query"` | 非交互式查询后退出         | `claude -p "explain this function"` |
| `claude -c`         | 继续当前目录最近一次会话      | `claude -c`                         |
| `claude -r`         | 恢复历史会话            | `claude -r`                         |
| `/help`             | 查看帮助              | `/help`                             |
| `/clear`            | 清理当前上下文           | `/clear`                            |
| `/init`             | 生成项目级 `CLAUDE.md` | `/init`                             |
| `/memory`           | 查看和编辑记忆文件         | `/memory`                           |
| `/permissions`      | 配置权限规则            | `/permissions`                      |
| `/plan`             | 进入计划模式            | `/plan`                             |
| `/agents`           | 管理子代理             | `/agents`                           |
| `/hooks`            | 查看 hooks          | `/hooks`                            |
| `/context`          | 查看上下文使用情况         | `/context`                          |
| `/compact`          | 压缩上下文             | `/compact`                          |
| `/diff`             | 查看本次改动            | `/diff`                             |
| `/code-review`      | 代码审查              | `/code-review`                      |

---

## 五、权限模式选择

Claude Code 的权限模式决定它在执行文件修改、命令运行、网络访问等操作时是否需要你确认。

| 模式                  | 特点           | 适用场景              | 风险          |
| ------------------- | ------------ | ----------------- | ----------- |
| `default`           | 默认只读，修改前确认   | 初次使用、敏感项目         | 操作较慢        |
| `acceptEdits`       | 可自动接受文件编辑    | 你会持续 review 的编码任务 | 可能误改文件      |
| `plan`              | 只读分析，不改代码    | 调研、方案设计、大改前分析     | 不会直接执行      |
| `auto`              | 减少审批，中间有安全检查 | 长任务、低风险任务         | 仍需审查最终 diff |
| `dontAsk`           | 只允许预先批准的工具   | CI、脚本化流程          | 配置成本较高      |
| `bypassPermissions` | 跳过权限检查       | 仅限隔离容器或虚拟机        | 高风险，不建议日常使用 |

推荐默认策略：

```text
陌生代码库：plan → default
日常小改动：acceptEdits
长时间批量任务：auto + 明确边界 + 测试门禁
CI 自动化：dontAsk + 白名单工具
高风险命令：永远人工确认
```

---

## 六、项目记忆：CLAUDE.md 使用规范

### 6.1 CLAUDE.md 是什么

`CLAUDE.md` 是 Claude Code 每次会话启动时读取的项目说明文件。它适合放长期有效、对项目行为有影响的信息。

适合写入：

* 项目启动命令
* 测试命令
* 构建命令
* 代码风格
* 分支命名规则
* PR 规则
* 特殊架构约定
* 容易踩坑的项目细节
* 不允许修改的目录或文件

不适合写入：

* 大段业务文档
* API 文档全文
* 文件逐个说明
* 经常变化的信息
* “写干净代码”这类空泛要求
* Claude 通过读代码即可知道的信息

### 6.2 推荐 CLAUDE.md 模板

```markdown
# Project Instructions for Claude Code

## Project Overview
- This project is a [type of project].
- Main language/framework: [stack].
- Main entry points: [files].

## Common Commands
- Install dependencies: `...`
- Run dev server: `...`
- Run unit tests: `...`
- Run typecheck: `...`
- Run lint: `...`
- Build: `...`

## Code Style
- Use [style rule].
- Follow existing patterns in nearby files.
- Do not introduce new dependencies without asking first.

## Testing Rules
- For bug fixes, first add or identify a failing test.
- Prefer running targeted tests before the full suite.
- Always show the exact test command and result.

## Git Workflow
- Create a new branch for non-trivial work.
- Keep commits focused and descriptive.
- Do not commit secrets, generated files, or unrelated formatting changes.

## Safety Rules
- Do not modify `.env`, secrets, credentials, migrations, deployment configs, or billing/payment code without explicit approval.
- Before deleting files or running destructive commands, ask for confirmation.
- If requirements are unclear, ask questions before editing.

## Definition of Done
- Code implemented.
- Relevant tests added or updated.
- Tests/typecheck/lint run where applicable.
- Diff summarized.
- Remaining risks listed.
```

### 6.3 CLAUDE.md 维护原则

* 控制长度，越短越容易被遵守。
* 每条规则都应该能减少实际错误。
* 发现 Claude 重复犯错时，更新 CLAUDE.md。
* 团队项目应把项目级 `CLAUDE.md` 提交到 Git。
* 个人偏好放入 `CLAUDE.local.md`，并加入 `.gitignore`。

---

## 七、高质量提示词公式

### 7.1 通用公式

```text
目标 + 范围 + 背景 + 限制 + 验证方式 + 输出格式 + 不确定项处理
```

模板：

```text
你正在处理一个 [技术栈] 项目。

任务目标：
[明确要完成什么]

范围：
- 请重点查看：[文件/目录]
- 不要修改：[文件/目录]

背景：
[现象、业务原因、相关上下文]

限制：
- 不要引入新依赖，除非先征得我同意。
- 不要改变现有 API 行为，除非明确说明兼容性影响。
- 保持现有代码风格。

验证方式：
- 请运行：[测试/构建/lint 命令]
- 如果测试无法运行，请说明原因和你已完成的替代验证。

输出要求：
1. 你修改了哪些文件
2. 为什么这样改
3. 验证结果
4. 剩余风险

如果有任何不确定项，先问我，不要猜测。
```

### 7.2 Bug 修复提示词

```text
请修复以下 bug，但先不要直接改代码。

现象：
[描述 bug]

复现步骤：
1. ...
2. ...
3. ...

期望行为：
[应该发生什么]

实际行为：
[现在发生什么]

要求：
1. 先定位可能相关的文件和根因。
2. 写出修复计划。
3. 等我确认后再实现。
4. 优先添加能复现该 bug 的测试。
5. 修复后运行相关测试，并展示命令和结果。
6. 不要通过跳过测试、吞掉异常、删除校验来掩盖问题。
```

### 7.3 新功能提示词

```text
我要实现一个新功能：[功能名称]

用户故事：
作为 [用户角色]，我希望 [能力]，以便 [价值]。

验收标准：
- [标准 1]
- [标准 2]
- [标准 3]

技术限制：
- 使用现有技术栈。
- 不新增第三方依赖，除非先说明理由。
- 遵循现有代码结构和命名风格。

请先完成：
1. 阅读相关代码。
2. 总结现有实现模式。
3. 提出实现方案。
4. 列出需要修改的文件。
5. 标明风险点和需要我确认的问题。

在我确认前，不要修改代码。
```

### 7.4 代码审查提示词

```text
请审查当前 diff。

重点关注：
1. 正确性 bug
2. 安全风险
3. 边界条件
4. 错误处理
5. 兼容性
6. 测试覆盖
7. 是否有无关改动

输出格式：
- Blocking issues：必须修复
- Suggestions：建议优化
- Questions：需要作者确认的问题
- Positive notes：做得好的地方

只提出有实际价值的问题，不要做泛泛的风格评论。
```

### 7.5 重构提示词

```text
请重构 [模块/文件]，目标是 [目标，例如降低复杂度、拆分职责、提升可读性]。

约束：
- 不改变外部行为。
- 不修改公共 API，除非先提出并获得确认。
- 每一步保持测试可通过。
- 避免一次性大规模改动。

流程：
1. 先解释当前结构和问题。
2. 给出分阶段重构计划。
3. 每阶段说明风险和验证方式。
4. 等我确认后逐步实施。
```

---

## 八、推荐工作流

### 8.1 新项目接手

```text
请作为资深工程师帮我快速熟悉这个代码库。
不要修改任何文件。

请输出：
1. 项目用途
2. 技术栈
3. 目录结构说明
4. 核心业务流程
5. 启动、测试、构建命令
6. 关键配置文件
7. 潜在风险点
8. 我应该优先阅读的 5 个文件
```

### 8.2 修复线上问题

```text
线上出现问题：[问题描述]

已知信息：
- 时间：...
- 影响范围：...
- 相关日志：...
- 最近变更：...

请按事故排查流程工作：
1. 先根据日志和代码推断可能根因。
2. 列出需要检查的文件和命令。
3. 不要直接改代码，先给排查计划。
4. 修复时优先最小改动。
5. 修复后给出验证步骤和回滚建议。
```

### 8.3 从 issue 到 PR

```text
请处理 GitHub issue #[编号]。

流程：
1. 使用 gh 查看 issue 详情。
2. 阅读相关代码。
3. 总结问题和验收标准。
4. 给出实现计划。
5. 等我确认后创建分支并修改代码。
6. 添加或更新测试。
7. 运行相关检查。
8. 提交 commit 并生成 PR 描述草稿。
```

### 8.4 让 Claude 自己先问你问题

适合需求还不清楚的大功能：

```text
我想实现：[一句话描述]

请先不要写代码。请像资深技术负责人一样采访我。
请重点追问：
1. 用户场景
2. 数据模型
3. 权限和安全
4. UI/UX 边界
5. 兼容性
6. 错误处理
7. 测试和验收标准
8. 是否有不该做的范围

一次最多问 7 个最关键的问题，并说明每个问题为什么重要。
```

---

## 九、上下文管理

Claude Code 的上下文会被对话、读取文件、命令输出、日志等快速占满。上下文越臃肿，越容易遗忘早期要求或产生错误。

### 9.1 控制上下文的做法

* 不要一次粘贴超长日志，先截取关键错误。
* 让 Claude 自己用 `grep`、`rg`、`git diff` 查需要的信息。
* 长任务拆分为多个阶段。
* 每个阶段结束后让 Claude 总结当前状态。
* 使用 `/context` 查看上下文使用情况。
* 使用 `/compact` 压缩长会话。
* 新任务和旧任务无关时，使用 `/clear` 或新开会话。

### 9.2 阶段总结提示词

```text
请总结当前任务状态，格式如下：
1. 已完成
2. 已修改文件
3. 验证结果
4. 尚未完成
5. 关键决策
6. 下一步建议
请写得简洁，方便后续压缩上下文或恢复会话。
```

---

## 十、Hooks：把“必须执行”变成自动化

`CLAUDE.md` 是指令，Claude 可能理解或忽略；Hooks 是确定性脚本，适合必须每次执行的规则。

适合用 hooks 的场景：

* 编辑后自动格式化
* 提交前运行 lint
* 禁止修改敏感目录
* Claude 需要输入时发送通知
* 每次启动注入上下文
* 对修改进行安全审计

示例：编辑后运行格式化。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $(jq -r '.tool_input.file_path')"
          }
        ]
      }
    ]
  }
}
```

使用建议：

* 高风险 hook 先在测试仓库验证。
* hook 命令应尽量短、可解释、可重复执行。
* 不要在 hook 中执行不可逆操作。
* 团队共享 hook 前应 code review。

---

## 十一、Skills：沉淀重复流程

当你反复粘贴同一套步骤、检查清单或团队规范时，应把它做成 Skill。

目录结构示例：

```text
.claude/skills/fix-issue/SKILL.md
```

示例：

```markdown
---
name: fix-issue
description: Analyze and fix a GitHub issue with tests and PR summary
---

Fix the GitHub issue: $ARGUMENTS.

Workflow:
1. Use `gh issue view` to read the issue.
2. Identify relevant files.
3. Reproduce the issue if possible.
4. Implement the smallest safe fix.
5. Add or update tests.
6. Run relevant checks.
7. Summarize changed files, test results, and residual risks.
```

调用：

```text
/fix-issue 1234
```

适合沉淀为 Skill 的内容：

* PR Review 流程
* 安全审查流程
* API 设计规范
* 数据库迁移检查
* 发布流程
* 客户问题排查流程
* 组件生成规范

---

## 十二、Subagents：让不同角色分工

子代理适合执行独立、专业、上下文较重的任务。它们可以拥有自己的提示词、工具限制和模型选择。

典型角色：

* `security-reviewer`：安全审查
* `test-writer`：补测试
* `frontend-reviewer`：前端交互和可访问性审查
* `db-migration-reviewer`：数据库迁移审查
* `performance-profiler`：性能分析
* `docs-writer`：文档整理

示例：

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer.
Review code for:
- SQL injection
- XSS
- command injection
- authentication flaws
- authorization flaws
- secrets in code
- insecure data handling

Provide specific file and line references, severity, and suggested fixes.
Do not modify files unless explicitly asked.
```

调用方式：

```text
Use the security-reviewer agent to review the current diff.
```

---

## 十三、MCP：连接外部工具和上下文

MCP 适合把 Claude Code 接入外部系统，例如：

* GitHub / GitLab
* Jira / Linear
* Notion / Google Drive
* Figma
* 数据库
* 监控系统
* 内部工具

使用原则：

* 只连接当前任务需要的工具。
* 区分个人级、项目级、团队级配置。
* 对生产数据库、客户数据、密钥系统保持最小权限。
* 给 Claude 明确说明数据边界和禁止操作。

示例提示：

```text
请通过 MCP 读取 Linear ticket 的需求说明，但不要修改 ticket 状态。
只根据 ticket 内容和当前代码库制定实现计划。
```

---

## 十四、Git 与 PR 工作流

### 14.1 改动前

```text
请先运行 git status 和 git diff，确认当前工作区状态。
不要覆盖我已有的未提交改动。
如果发现无关改动，请先告诉我。
```

### 14.2 改动后

```text
请总结当前 diff：
1. 修改了哪些文件
2. 每个文件为什么修改
3. 行为变化
4. 测试结果
5. 风险和需要 reviewer 重点看的地方
```

### 14.3 提交

```text
请创建一个描述性 commit。
commit message 使用 Conventional Commits 风格。
不要提交无关文件。
提交前先展示 git diff 摘要。
```

### 14.4 PR 描述模板

```markdown
## Summary
- ...

## Changes
- ...

## Testing
- [ ] Unit tests
- [ ] Typecheck
- [ ] Lint
- [ ] Manual verification

## Risks
- ...

## Reviewer Notes
- ...
```

---

## 十五、GitHub Actions 自动化

Claude Code 可以进入 GitHub PR / Issue 工作流，用于：

* 通过 `@claude` 触发分析
* 根据 issue 创建 PR
* 自动修复简单问题
* 代码审查
* CI 中执行自定义任务

推荐用法：

```text
@claude please review this PR for correctness bugs, missing tests, and risky edge cases. Do not make changes; leave review comments only.
```

或：

```text
@claude implement this issue. Follow CLAUDE.md, add tests, and open a PR with a clear summary and testing notes.
```

安全建议：

* 不要给 CI 中的 Claude 过大的权限。
* 对外部贡献者 PR 保持更严格限制。
* 保护 secret，不允许 Claude 输出或修改密钥。
* 自动化修复必须经过人类 review。
* 对生产部署相关操作使用手动审批。

---

## 十六、安全红线

Claude Code 可以运行命令和修改文件，因此必须设定边界。

### 16.1 高风险操作必须人工确认

包括：

* 删除文件或目录
* 修改数据库迁移
* 运行生产环境命令
* 修改 `.env`、密钥、凭证
* 更改鉴权、支付、账单、权限逻辑
* 修改 CI/CD 部署流程
* 升级核心依赖
* 大规模格式化全仓库
* force push / reset / rebase 公共分支

### 16.2 推荐安全提示词

```text
在本任务中，请遵守以下安全边界：
1. 不要修改 secrets、.env、生产配置或部署文件。
2. 不要删除文件，除非先列出原因并获得我确认。
3. 不要运行 destructive 命令，例如 rm -rf、reset --hard、force push。
4. 不要新增依赖，除非先说明必要性、替代方案和风险。
5. 对鉴权、支付、权限相关代码只提出计划，不要直接修改，除非我确认。
```

---

## 十七、质量门禁清单

每次让 Claude Code 完成代码任务后，都要求它输出：

```text
请按以下格式做最终报告：

## Done
- ...

## Files Changed
| File | Change | Reason |
|---|---|---|

## Verification
| Check | Command | Result |
|---|---|---|

## Risks
- ...

## Follow-ups
- ...
```

人工 review 时重点检查：

* 是否修改了无关文件
* 是否绕过测试或校验
* 是否引入过度抽象
* 是否新增不必要依赖
* 是否破坏兼容性
* 是否处理错误分支和边界条件
* 是否暴露密钥或敏感日志
* 是否提交了生成文件、缓存文件、临时文件

---

## 十八、常见错误与修正

| 错误用法              | 问题        | 改进方式                   |
| ----------------- | --------- | ---------------------- |
| “帮我重构整个项目”        | 范围过大      | 先让它分析并拆阶段              |
| “修复 bug”          | 缺少复现和成功标准 | 提供现象、日志、期望行为、测试要求      |
| 一次粘贴 5000 行日志     | 上下文浪费     | 先贴核心错误，让 Claude 用命令查日志 |
| 盲目 Accept All     | 容易误改      | 定期 `/diff`，小步 review   |
| CLAUDE.md 写成百科    | 规则被淹没     | 保留关键规则和命令              |
| 让 Claude 直接改高风险代码 | 容易出事故     | plan 模式 + 人工确认         |
| 不运行测试             | 只能“看起来完成” | 每次都给验证命令               |
| 不看最终 diff         | 难发现无关改动   | 改完必须总结 diff            |

---

## 十九、推荐的团队落地规范

团队使用 Claude Code 时，建议建立以下文件：

```text
CLAUDE.md
.claude/settings.json
.claude/skills/
.claude/agents/
docs/ai-workflow.md
```

团队规范至少包括：

* 允许 Claude 自动修改的目录
* 禁止 Claude 自动修改的目录
* 标准测试命令
* 标准 PR 描述模板
* 代码审查要求
* 依赖新增审批规则
* 安全边界
* 常用 Skills
* 推荐 Subagents
* CI 中 Claude 的权限范围

---

## 二十、最佳实践速查卡

### 20.1 每次任务开始前

```text
先读代码，不改文件。
先给计划，不直接实现。
先问不确定项，不要猜。
```

### 20.2 每次编码时

```text
小步修改。
遵循现有模式。
不新增无关依赖。
不做无关重构。
```

### 20.3 每次完成后

```text
运行测试。
展示命令和结果。
总结 diff。
列出风险。
等待人类 review。
```

### 20.4 最推荐的一条提示词

```text
请像资深工程师一样处理这个任务。
先阅读相关代码并制定计划，不要立刻修改。
如果需求、边界、测试方式或风险点有任何不确定，请先问我。
实现时保持最小改动，遵循现有代码风格。
完成后运行相关测试，展示命令和结果，并总结所有修改和剩余风险。
```

---

## 二十一、终极原则

Claude Code 的价值不在于让你完全退出开发流程，而在于把你从低价值重复劳动中解放出来，让你专注于目标、边界、架构判断、风险控制和最终审查。

最佳心法：

```text
把 Claude Code 当作一名速度很快、需要清晰任务边界、需要测试约束、需要 code review 的初级到中高级工程师。
```

你给的上下文越清楚，它越可靠；你给的验证越明确，它越接近可交付；你设的边界越严谨，它越安全。
