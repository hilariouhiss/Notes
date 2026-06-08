# Git 命令小全

> 一份覆盖日常开发、团队协作与救急场景的实用参考手册。

## 一、 Git 简介

**Git** 是 Linus Torvalds 于 2005 年创建的一款**分布式版本控制系统**（DVCS）。与集中式版本控制（如 SVN）不同，Git 在每个开发者的本地都保存一份完整的代码仓库副本，包含全部历史记录，因此绝大多数操作都可以在本地离线完成，速度极快。

**核心特点**：

| 特性 | 说明 |
|------|------|
| **分布式** | 每个节点都有完整仓库，不依赖单一服务器 |
| **快照机制** | 记录文件的完整快照，而非增量差异 |
| **分支轻量** | 分支创建与切换几乎瞬时完成，鼓励频繁分支 |
| **数据完整性** | 基于 SHA-1 校验和，确保历史记录不可篡改 |
| **Stage 区域** | 引入暂存区（Index），可精确控制每次提交的内容 |

**适用场景**：个人项目管理、团队协作开发、开源贡献、代码版本追溯、持续集成流水线等。

## 二、 配置命令

### 2.1 基础配置

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git config --global user.name "Your Name"` | 设置全局用户名 | 初次安装 Git 后必做 |
| `git config --global user.email "email@example.com"` | 设置全局邮箱 | 初次安装 Git 后必做，需与远程平台一致 |
| `git config --global core.editor "vim"` | 设置默认文本编辑器 | 需要编写多行提交信息时 |
| `git config --global init.defaultBranch main` | 设置默认分支名 | 新仓库默认使用 `main` 而非 `master` |

### 2.2 查看与修改配置

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git config --list` | 列出所有配置项 | 检查当前环境配置 |
| `git config user.name` | 查看指定配置项 | 快速确认用户名/邮箱 |
| `git config --global --edit` | 编辑全局配置文件 | 批量修改配置项 |
| `git config --local user.name "Name"` | 仅对当前仓库设置 | 工作项目与个人项目使用不同身份 |

### 2.3 实用配置推荐

```bash
# 让 Git 输出更简洁
$ git config --global format.pretty oneline

# 设置命令别名（大幅提高日常效率）
$ git config --global alias.st status
$ git config --global alias.co checkout
$ git config --global alias.br branch
$ git config --global alias.ci commit
$ git config --global alias.lg "log --oneline --graph --decorate"

# 开启颜色输出
$ git config --global color.ui auto
```

## 三、 仓库操作

### 3.1 创建与克隆

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git init` | 在当前目录初始化新仓库 | 从零开始一个新项目 |
| `git init <directory>` | 在指定目录初始化新仓库 | 为尚未使用版本控制的项目创建仓库 |
| `git clone <url>` | 克隆远程仓库到本地 | 参与已有项目，下载完整历史 |
| `git clone <url> <dir>` | 克隆到指定目录名 | 需要自定义本地文件夹名称时 |
| `git clone --depth 1 <url>` | 浅克隆（仅最新提交） | 仅需查看最新代码，大幅节省时间与空间 |

**常见用法**：

```bash
# 克隆 GitHub 仓库
$ git clone https://github.com/user/repo.git

# 克隆并指定分支
$ git clone -b develop https://github.com/user/repo.git

# 使用 SSH 克隆（需配置密钥）
$ git clone git@github.com:user/repo.git
```

## 四、 日常 Workflow

### 4.1 查看状态与差异

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git status` | 查看工作区与暂存区状态 | **最常用**，提交前确认改动范围 |
| `git status -s` | 简洁格式状态 | 快速浏览大量文件变更 |
| `git diff` | 查看工作区 vs 暂存区的差异 | 提交前审查具体修改内容 |
| `git diff --cached` | 查看暂存区 vs 最新提交的差异 | 确认已暂存的内容是否准备就绪 |
| `git diff HEAD` | 查看工作区 vs 最新提交的差异 | 查看所有未提交的改动总和 |
| `git diff <file>` | 查看指定文件的差异 | 聚焦审查单个文件 |

### 4.2 添加与暂存

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git add <file>` | 将指定文件添加到暂存区 | 精确控制每次提交的文件 |
| `git add <dir>/` | 添加整个目录 | 批量添加一个模块的改动 |
| `git add .` | 添加当前目录所有变更 | 快速暂存所有改动（需确认 `status`） |
| `git add -A` | 添加所有变更（含删除） | 跨目录的全面提交前使用 |
| `git add -p` | 交互式选择代码块 | **最佳实践**，逐块审查并选择需要暂存的部分 |

**常见用法**：

```bash
# 交互式暂存（强烈推荐养成此习惯）
$ git add -p
# 然后根据提示选择: y(添加) n(跳过) s(拆分) e(编辑)
```

### 4.3 提交更改

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git commit -m "message"` | 提交暂存区的改动 | 常规提交，附带简短描述 |
| `git commit -a -m "message"` | 跳过暂存，直接提交已跟踪文件 | 快速修改后提交，但需谨慎 |
| `git commit --amend` | 修改最后一次提交 | 提交后发现遗漏文件或需要修改提交信息 |
| `git commit --amend --no-edit` | 追加到上次提交，不改信息 | 上次提交后立即发现漏了文件 |

**常见用法**：

```bash
# 修改最后一次提交（还未 push 到远程时）
$ git add forgotten-file.js
$ git commit --amend --no-edit

# 修改提交信息
$ git commit --amend -m "新的提交信息"
```

### 4.4 删除与重命名

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git rm <file>` | 删除文件并从 Git 跟踪中移除 | 彻底删除不再需要的文件 |
| `git rm --cached <file>` | 仅从 Git 跟踪中移除，保留本地文件 | 误将配置文件/敏感文件提交后移除跟踪 |
| `git mv <old> <new>` | 重命名文件 | Git 会识别为 rename 而非 delete+add |

## 五、 分支管理

### 5.1 查看分支

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git branch` | 列出本地分支 | 快速查看当前有哪些分支 |
| `git branch -v` | 显示分支与最后提交 | 查看各分支的最新状态 |
| `git branch -r` | 列出远程分支 | 了解远程仓库有哪些分支 |
| `git branch -a` | 列出所有本地与远程分支 | 全面查看项目分支结构 |
| `git branch --merged` | 显示已合并到当前分支的分支 | 清理已完成的特性分支 |
| `git branch --no-merged` | 显示尚未合并的分支 | 避免误删进行中的分支 |

### 5.2 创建与切换分支

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git branch <name>` | 创建新分支（不切换） | 基于当前 HEAD 创建分支 |
| `git checkout <branch>` | 切换到指定分支 | 切换到已有分支继续工作 |
| `git checkout -b <name>` | 创建并切换到新分支 | **最常用**，开始新特性开发 |
| `git switch <branch>` | 切换到分支（Git 2.23+ 推荐） | 语义更清晰的分支切换 |
| `git switch -c <name>` | 创建并切换（Git 2.23+） | 替代 `checkout -b` |
| `git checkout -b <name> <base>` | 基于指定提交/分支创建 | 从特定节点创建特性分支 |

**常见用法**：

```bash
# 开始新特性开发
$ git switch -c feature/login

# 基于远程分支创建本地分支
$ git checkout -b feature/api origin/feature/api

# 创建修复分支
$ git switch -c hotfix/critical-bug main
```

### 5.3 合并分支

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git merge <branch>` | 将指定分支合并到当前分支 | 特性开发完成后合并到主分支 |
| `git merge --no-ff <branch>` | 禁用快进合并，保留分支历史 | 需要明确看到特性分支的合并记录 |
| `git merge --squash <branch>` | 将分支所有提交压缩为一个 | 特性分支提交较凌乱，合并为单一提交 |

**常见用法**：

```bash
# 合并特性分支到 main（推荐流程）
$ git switch main
$ git pull origin main
$ git merge feature/login
$ git push origin main
```

### 5.4 解决合并冲突

```bash
# 当 merge 出现冲突时
$ git status                    # 查看冲突文件列表
# 手动编辑冲突文件，保留需要的代码
$ git add <resolved-file>       # 标记冲突已解决
$ git commit                    # 完成合并提交
```

**冲突标记格式**：

```text
<<<<<<< HEAD
当前分支的代码
=======
合并分支的代码
>>>>>>> feature/branch
```

### 5.5 变基（Rebase）

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git rebase <branch>` | 将当前分支变基到指定分支 | 保持线性提交历史 |
| `git rebase -i HEAD~3` | 交互式变基最近 3 个提交 | **整理提交历史**：合并、修改、删除、重排提交 |
| `git rebase --continue` | 解决冲突后继续变基 | 变基过程中冲突解决后 |
| `git rebase --abort` | 取消正在进行中的变基 | 变基出错时安全回退 |

**常见用法**：

```bash
# 交互式整理提交历史（feature 分支 push 前）
$ git rebase -i HEAD~5
# 可执行操作: pick(保留) reword(改信息) squash(合并) drop(删除) reoder(重排)

# 将特性分支变基到最新的 main
$ git switch feature/login
$ git rebase main
```

**⚠️ 重要原则**：**不要对已经推送到远程的提交进行 rebase**，这会改写历史导致团队成员困扰。

## 六、 远程协作

### 6.1 远程仓库管理

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git remote -v` | 查看已配置的远程仓库 | 确认关联的远程地址 |
| `git remote add <name> <url>` | 添加远程仓库 | 关联 GitHub/GitLab 等平台仓库 |
| `git remote rename <old> <new>` | 重命名远程仓库 | 远程仓库名称变更时 |
| `git remote remove <name>` | 删除远程仓库关联 | 不再需要的远程仓库 |
| `git remote set-url <name> <url>` | 修改远程仓库地址 | 协议切换（HTTPS ↔ SSH）或地址变更 |

### 6.2 推送代码

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git push <remote> <branch>` | 推送本地分支到远程 | 日常提交代码到远程 |
| `git push -u <remote> <branch>` | 推送并设置上游分支 | **首次推送新分支**，建立跟踪关系 |
| `git push` | 推送到已设置上游的分支 | 建立上游后简写推送 |
| `git push --force` | 强制推送（⚠️ 慎用） | 本地 rebase 后需要覆盖远程（确保无人基于该分支工作） |
| `git push --force-with-lease` | 安全的强制推送 | 比 `--force` 更安全，远程有新提交时会拒绝 |
| `git push <remote> --delete <branch>` | 删除远程分支 | 特性分支合并后清理远程分支 |

**常见用法**：

```bash
# 首次推送新分支
$ git push -u origin feature/login

# 之后简写推送
$ git push

# 安全强制推送（rebase 后）
$ git push --force-with-lease origin feature/login

# 删除远程分支
$ git push origin --delete feature/old-feature
```

### 6.3 拉取与同步

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git fetch <remote>` | 从远程下载最新变更（不合并） | **推荐做法**：先查看远程变更再决定如何整合 |
| `git fetch --all` | 从所有远程获取最新变更 | 多远程仓库项目 |
| `git pull <remote> <branch>` | 拉取并合并远程变更 | 快速同步远程最新代码 |
| `git pull --rebase` | 拉取并以 rebase 方式整合 | 保持本地提交历史的线性整洁 |

**常见用法**：

```bash
# 推荐的安全同步流程
$ git fetch origin
$ git log HEAD..origin/main --oneline   # 查看远程有哪些新提交
$ git merge origin/main                  # 或 rebase

# 简洁但风险较高的方式
$ git pull origin main

# 保持线性历史
$ git pull --rebase origin main
```

### 6.4 拉取远程分支

```bash
# 获取远程分支并在本地创建跟踪分支
$ git fetch origin
$ git switch -c feature/remote origin/feature/remote

# 或简写（Git 2.23+）
$ git switch feature/remote    # 自动建立跟踪
```

## 七、 查看历史与追溯

### 7.1 提交日志

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git log` | 查看提交历史 | 浏览项目完整历史 |
| `git log --oneline` | 一行显示每个提交 | 快速浏览大量历史 |
| `git log --graph` | 图形化显示分支合并 | 理解分支结构 |
| `git log --decorate` | 显示分支与标签指针 | 查看各分支 HEAD 位置 |
| `git log -n 5` | 仅显示最近 5 条 | 查看最新动态 |
| `git log --all` | 显示所有分支的历史 | 全局视角查看项目 |
| `git log --author="name"` | 按作者过滤 | 查看特定成员的贡献 |
| `git log --since="2024-01-01"` | 按时间过滤 | 查看特定时间段的历史 |
| `git log --grep="fix"` | 按提交信息关键词过滤 | 查找特定类型的提交 |
| `git log -- <file>` | 查看指定文件的历史 | 追溯文件的变更过程 |
| `git log -p -- <file>` | 查看文件的详细修改历史 | 代码审查、定位 bug 引入 |

**推荐组合别名**：

```bash
# 效果极佳的日志视图
$ git log --oneline --graph --decorate --all
# 可配置为别名: git config --global alias.lga "log --oneline --graph --decorate --all"
```

### 7.2 查看具体提交

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git show <commit>` | 显示某提交的详细信息和 diff | 查看特定提交改了什么 |
| `git show <commit>:<file>` | 查看某提交中文件的内容 | 追溯文件在某个版本的状态 |
| `git show --stat <commit>` | 显示提交的文件改动统计 | 快速了解提交的影响范围 |

### 7.3 代码追溯

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git blame <file>` | 逐行显示最后修改的作者和提交 | **定位 bug 责任人**、理解代码上下文 |
| `git blame -L 10,20 <file>` | 仅查看文件的第 10-20 行 | 聚焦特定代码段的修改历史 |


## 八、 撤销与恢复

### 8.1 工作区撤销（未暂存）

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git restore <file>` | 丢弃工作区的修改 | 实验性修改后决定放弃（Git 2.23+） |
| `git checkout -- <file>` | 同上（旧版本命令） | 兼容旧版 Git |

**⚠️ 注意**：此操作会**永久丢失**工作区的修改，执行前请确认。

### 8.2 暂存区撤销（已暂存未提交）

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git restore --staged <file>` | 将文件从暂存区移回工作区 | 误 `git add` 了不该提交的文件 |
| `git reset HEAD <file>` | 同上（旧版本命令） | 兼容旧版 Git |

### 8.3 提交撤销（已提交）

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git reset --soft HEAD~1` | 撤销最后一次提交，改动保留在暂存区 | 需要修改提交信息或重新组织暂存内容 |
| `git reset --mixed HEAD~1` | 撤销最后一次提交，改动保留在工作区 | 需要重新选择要提交的内容（默认模式） |
| `git reset --hard HEAD~1` | 彻底丢弃最后一次提交及改动 | **⚠️ 危险**：确认不需要这些改动 |
| `git revert <commit>` | 创建一个新提交来撤销指定提交的改动 | **最安全的方式**，用于已推送的提交 |

**常见用法**：

```bash
# 撤销最后一次提交（保留改动）
$ git reset --soft HEAD~1

# 撤销某个历史提交（推荐用于公共分支）
$ git revert abc1234
$ git push

# 回退到指定版本（⚠️ 本地开发分支）
$ git reset --hard abc1234
```

### 8.4 找回丢失的提交

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git reflog` | 查看 HEAD 的所有变动记录 | **救命命令**：找回 reset/rebase 丢失的提交 |
| `git fsck --lost-found` | 查找悬空对象 | `reflog` 也无法找回时的最后手段 |

**常见用法**：

```bash
# 找回误删的提交
$ git reflog
# 找到需要的 commit hash（如 abc1234）
$ git checkout abc1234          # 查看确认
$ git switch -c recovery-branch  # 基于该提交创建分支
```

## 九、 临时存储（Stash）

Stash 适用于：临时切换到其他分支处理紧急任务，但当前分支的改动还不适合提交。

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git stash` | 暂存当前所有改动（含暂存区） | 临时切换分支前保存当前工作 |
| `git stash push -m "描述"` | 带描述地暂存 | 多条 stash 时便于区分 |
| `git stash list` | 查看所有暂存记录 | 管理多个 stash |
| `git stash pop` | 恢复最近的 stash 并从列表删除 | **最常用**：完成任务后恢复工作 |
| `git stash apply` | 恢复最近的 stash 但保留在列表 | 需要应用到多个分支时 |
| `git stash apply stash@{1}` | 恢复指定的 stash | 恢复某条特定的 stash |
| `git stash drop stash@{0}` | 删除指定的 stash | 清理不需要的 stash |
| `git stash clear` | 清除所有 stash | 批量清理 |
| `git stash show -p` | 查看最近 stash 的详细 diff | 恢复前确认 stash 内容 |

**常见用法**：

```bash
# 紧急修复流程
$ git stash push -m "half-done feature"
$ git switch -c hotfix/urgent main
# ... 修复代码，提交，合并 ...
$ git switch feature/my-feature
$ git stash pop
# 继续之前的工作
```

## 十、 标签管理

标签用于标记重要的里程碑（如版本发布）。

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git tag` | 列出所有标签 | 查看项目有哪些版本标记 |
| `git tag <name>` | 创建轻量标签 | 临时标记 |
| `git tag -a <name> -m "message"` | 创建附注标签 | **推荐**：正式发布版本 |
| `git tag -a <name> <commit>` | 为历史提交补打标签 | 发布后忘记打标签 |
| `git show <tag>` | 查看标签信息与对应提交 | 查看某个版本的详情 |
| `git push <remote> <tag>` | 推送指定标签到远程 | 发布某个版本 |
| `git push <remote> --tags` | 推送所有本地标签 | 批量同步标签到远程 |
| `git tag -d <name>` | 删除本地标签 | 误打的标签 |

**常见用法**：

```bash
# 发布新版本
$ git tag -a v1.2.0 -m "Release version 1.2.0"
$ git push origin v1.2.0

# 为过去的提交补打标签
$ git log --oneline
$ git tag -a v1.1.1 abc1234 -m "Patch release 1.1.1"
```

## 十一、 高级操作

### 11.1 拣选提交（Cherry-pick）

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git cherry-pick <commit>` | 将指定提交应用到当前分支 | 把某个分支的特定提交迁移到当前分支 |
| `git cherry-pick -x <commit>` | 拣选并保留原始提交信息 | 跨分支同步补丁时追溯来源 |
| `git cherry-pick <a>^..<b>` | 拣选一个范围的提交 | 批量迁移连续的提交 |

**常见用法**：

```bash
# 将 hotfix 分支的某个修复同步到 main
$ git switch main
$ git cherry-pick abc1234

# 将 feature 分支的一段提交拣选到 release 分支
$ git cherry-pick def5678^..ghi9012
```

### 11.2 交互式 Rebase 整理提交

```bash
# 整理最近 4 个提交
$ git rebase -i HEAD~4

# 常用操作指令：
# pick    - 保留提交（可重排顺序）
# reword  - 保留提交但修改提交信息
# squash  - 合并到上一个提交，保留提交信息
# fixup   - 合并到上一个提交，丢弃提交信息
# drop    - 删除提交
# edit    - 暂停在这里修改提交内容
```

### 11.3 子模块管理

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git submodule add <url> <path>` | 添加子模块 | 在主项目中引用另一个仓库 |
| `git submodule update --init` | 初始化并更新子模块 | 克隆包含子模块的项目后 |
| `git clone --recurse-submodules <url>` | 递归克隆含子模块的项目 | 一次性完整克隆 |

### 11.4 归档与清理

| 命令 | 说明 | 适用场景 |
|------|------|----------|
| `git archive -o backup.zip HEAD` | 将当前版本打包为压缩文件 | 交付不包含 `.git` 的源码包 |
| `git clean -n` | 预览将被清理的未跟踪文件 | 确认哪些文件会被删除 |
| `git clean -f` | 强制删除未跟踪文件 | 清理编译产物与临时文件 |
| `git clean -fd` | 删除未跟踪文件和目录 | 彻底清理 |


## 十二、 团队协作工作流

### 12.1 Git Flow 工作流

```bash
# 开始新特性
$ git switch -c feature/new-feature develop
# ... 开发与提交 ...
$ git switch develop
$ git merge --no-ff feature/new-feature
$ git push origin develop

# 发布新版本
$ git switch -c release/1.2.0 develop
# ... 版本准备 ...
$ git switch main
$ git merge --no-ff release/1.2.0
$ git tag -a v1.2.0
$ git push origin main --tags

# 紧急修复
$ git switch -c hotfix/critical-bug main
# ... 修复 ...
$ git switch main && git merge --no-ff hotfix/critical-bug
$ git switch develop && git merge --no-ff hotfix/critical-bug
```

### 12.2 GitHub Flow / Trunk-based 工作流

```bash
# 从 main 创建特性分支
$ git switch -c feature/my-feature main

# 开发与推送
$ git add .
$ git commit -m "feat: add new feature"
$ git push -u origin feature/my-feature

# 创建 Pull Request（在平台上完成代码审查）
# 审查通过后合并到 main
$ git switch main
$ git pull origin main
```

### 12.3 同步上游仓库（Fork 项目）

```bash
# 添加上游仓库
$ git remote add upstream https://github.com/original/repo.git

# 同步上游变更
$ git fetch upstream
$ git switch main
$ git merge upstream/main
$ git push origin main
```

更多工作流说明请查看 [[GitLab 完整工作流操作手册]]
## 十三、 救急

### 13.1 常见事故处理

| 问题 | 解决方案 |
|------|----------|
| **提交到了错误的分支** | `git reset HEAD~1 --soft` → 切换到正确分支 → 重新提交 |
| **提交了敏感信息** | 使用 [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) 或 `git filter-repo` 从历史中移除 |
| **误删了本地分支** | `git reflog` 找到分支最后的 commit → `git checkout -b <branch> <commit>` |
| **误删了远程分支** | 如果有同事克隆过，可让其推送；否则查看 CI/CD 或备份 |
| **代码冲突不会解** | `git merge --abort` 取消合并 → 理解双方改动后重试 |
| **rebase 出现冲突不想继续** | `git rebase --abort` 安全回退 |
| **push 被 rejected（non-fast-forward）** | 先 `git pull --rebase` 同步远程变更后再 push |
| **大文件误提交** | 从暂存区移除后用 `git commit --amend` |

### 13.2 紧急逃生命令

```bash
# 万能撤销（只要没 push，大部分操作可恢复）
$ git reflog                    # 找到操作前的 commit hash
$ git reset --hard <commit>      # 回退到该状态

# 彻底放弃一切本地修改（⚠️ 不可恢复）
$ git reset --hard HEAD
$ git clean -fd

# 仅放弃工作区修改，保留暂存区
$ git checkout -- .
```

## 十四、 提交信息规范（Conventional Commits）

良好的提交信息提高代码审查效率，也便于自动生成 CHANGELOG。

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat:` | 新功能 | `feat: add user authentication` |
| `fix:` | Bug 修复 | `fix: resolve null pointer in login` |
| `docs:` | 文档更新 | `docs: update API documentation` |
| `style:` | 代码格式（不影响功能） | `style: format with prettier` |
| `refactor:` | 代码重构 | `refactor: simplify data validation` |
| `perf:` | 性能优化 | `perf: optimize database queries` |
| `test:` | 测试相关 | `test: add unit tests for auth` |
| `chore:` | 构建/工具/依赖 | `chore: update dependencies` |
| `ci:` | 持续集成配置 | `ci: add GitHub Actions workflow` |
更多分支命名和提交规范见：[[Git 分支命名规范]]

## 十五、 附录：快速参考图

```
工作区 (Working Directory)
    │  git add
    ▼
暂存区 (Staging Area/Index)
    │  git commit
    ▼
本地仓库 (Local Repository)
    │  git push
    ▼
远程仓库 (Remote Repository)
    ▲
    │  git pull / git fetch
本地仓库
    ▲
    │  git checkout / git reset
暂存区
    ▲
    │  git restore --staged
工作区
```
