# GitLab 完整工作流操作手册

## 一、 前置准备

### 1.1 安装 Git
```bash
# 检查是否已安装
git --version

# 未安装时（以 Ubuntu 为例）
sudo apt update && sudo apt install git
```

### 1.2 配置身份信息
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### 1.3 配置 SSH Key（可选但推荐）
```bash
# 生成密钥（一路回车即可）
ssh-keygen -t ed25519 -C "your.email@example.com"

# 复制公钥内容
cat ~/.ssh/id_ed25519.pub

# 将内容粘贴到 GitLab → User Settings → SSH Keys → Add new key
```

## 二、 Fork 与 Clone

### 2.1 Fork 仓库（针对他人仓库）

**场景**：你需要向一个没有直接写入权限的仓库贡献代码。

1. 打开目标 GitLab 仓库页面
2. 点击右上角的 **Fork** 按钮
3. 选择你的个人命名空间（或组）
4. 等待 Fork 完成，此时你拥有了一个独立的副本

### 2.2 Clone 仓库到本地

**如果是自己创建的仓库或有写入权限的仓库**：
```bash
git clone git@192.168.101.173:original-group/project.git
```

### 2.3 配置上游远程（Upstream）

Fork 的仓库需要关联原始仓库，以便后续同步更新：

```bash
# 查看当前远程
git remote -v

# 添加上游远程（仅 Fork 场景需要）
git remote add upstream git@192.168.101.173:original-group/project.git

# 验证
git remote -v
# 应显示：
# origin    git@192.168.101.173:your-username/project.git (fetch)
# origin    git@192.168.101.173:your-username/project.git (push)
# upstream  git@192.168.101.173:original-group/project.git (fetch)
# upstream  git@192.168.101.173:original-group/project.git (push)
```

## 三、 本地开发工作流

### 3.1 同步主分支（开始工作前）

```bash
# 切换到 main 分支
git checkout main

# 拉取最新代码
git pull origin main
```

### 3.2 新建功能分支

**命名规范**：`feature/功能描述`、`bugfix/问题描述`、`hotfix/紧急修复`。

更多分支命名和提交规范见：[[Git 分支命名规范]]

```bash
# 基于最新的 main 创建分支
git checkout -b feature/login-module

# 查看当前分支
git branch
# * feature/login-module
#   main
```

## 四、 提交与推送

### 4.1 修改代码

正常编辑文件，使用 IDE 或文本编辑器进行开发。

### 4.2 查看变更

```bash
# 查看修改状态
git status

# 查看具体差异
git diff
```

### 4.3 暂存与提交

```bash
# 添加指定文件
git add src/login.cpp

# 或添加所有修改
git add .

# 提交（遵循 Commit Message 规范）
git commit -m "feat: 添加用户登录模块

- 实现 JWT Token 验证
- 添加登录状态持久化
- 修复密码加密逻辑

Closes #123"
```

### 4.4 推送分支到远程

```bash
# 首次推送（建立远程分支关联）
git push -u origin feature/login-module

# 后续推送
git push
```

## 五、 创建 Merge Request

### 5.1 在 GitLab Web 界面操作

1. 推送后，GitLab 页面会显示提示：**"Create merge request"**，点击进入
2. 或手动进入项目页面 → **Merge requests** → **New merge request**
3. 选择源分支：`feature/login-module`
4. 选择目标分支：`main`（或上游仓库的 `main`，如果是 Fork）

### 5.2 填写 MR 信息

| 字段            | 填写建议                                                |
| --------------- | ------------------------------------------------------- |
| **Title**       | 简洁描述，如 `feat: 添加用户登录模块`                   |
| **Description** | 详细说明变更内容、测试方法、关联 Issue（`Closes #123`） |
| **Assignee**    | 指定审查人                                              |
| **Reviewer**    | 指定代码审查者                                          |
| **Milestone**   | 关联里程碑（如有）                                      |
| **Labels**      | 标记类型（`feature`、`bugfix` 等）                      |

### 5.3 创建 Draft MR（可选）

如果代码尚未完成，但想提前收集反馈：
- 标题前缀加 `Draft:` 或 `WIP:`
- 完成后编辑标题去掉前缀，转为正式 MR

## 六、 代码审查与迭代

### 6.1 审查人进行 Code Review

审查人在 GitLab MR 页面：
- 查看 **Changes** 标签页的代码差异
- 点击行号添加 **行级评论**
- 提交 **General comment** 整体评价
- 选择 **Approve** 或 **Request changes**

### 6.2 开发者处理反馈

```bash
# 确保在功能分支上
git checkout feature/login-module

# 修改代码后，再次提交
git add .
git commit -m "fix: 根据审查意见调整密码加密强度"

# 推送（会自动更新到已有的 MR 中）
git push
```

**注意**：若审查要求修改历史提交（如合并多个小提交）：

```bash
# 交互式变基最近 3 个提交
git rebase -i HEAD~3

# 若已推送到远程，需强制推送（谨慎使用）
git push --force-with-lease
```

## 七、 合并到 main

### 7.1 合并前检查

确保满足以下条件：
- [ ] 所有 CI/CD Pipeline 通过（绿色对勾）
- [ ] 至少 1 位（或项目规定的）Reviewer 批准
- [ ] 无未解决的讨论（Unresolved discussions）
- [ ] 分支与 `main` 无冲突（或已解决）

### 7.2 执行合并

在 GitLab MR 页面点击 **Merge** 按钮，选择合并策略：

| 策略                   | 说明                               | 适用场景           |
| ---------------------- | ---------------------------------- | ------------------ |
| **Merge commit**       | 保留所有提交历史，生成一个合并提交 | 需要完整追溯历史   |
| **Squash and merge**   | 压缩所有提交为一个，再合并         | 功能分支提交较杂乱 |
| **Fast-forward merge** | 无合并提交，线性历史               | 要求提交历史整洁   |

**推荐**：功能开发使用 **Squash and merge**，保留清晰的 `main` 历史。

### 7.3 合并后

- GitLab 自动关闭关联的 Issue（如描述中有 `Closes #123`）
- 可选择 **Delete source branch** 自动删除远程功能分支

## 八、 同步与清理

### 8.1 更新本地 main

```bash
# 切换回 main
git checkout main

# 拉取最新合并后的代码
git pull origin main
# 或有 upstream 时：git pull upstream main
```

### 8.2 删除本地功能分支

```bash
# 删除已合并的本地分支
git branch -d feature/login-module

# 若分支未合并但需强制删除
git branch -D feature/login-module
```

### 8.3 删除远程功能分支（若未自动删除）

```bash
git push origin --delete feature/login-module
```

### 8.4 开始下一个功能

```bash
git checkout main
git pull origin main
git checkout -b feature/next-module
```

## 九、 附录：常见问题速查

| 问题             | 解决命令                                      |
| ---------------- | --------------------------------------------- |
| 撤销 `git add`   | `git restore --staged <file>`                 |
| 修改最后一次提交 | `git commit --amend`                          |
| 查看提交历史     | `git log --oneline --graph`                   |
| 暂存当前工作     | `git stash` / `git stash pop`                 |
| 解决合并冲突     | 手动编辑冲突文件 → `git add .` → `git commit` |
| 撤销本地所有修改 | `git restore .`（谨慎）                       |
| 查看分支图       | `git log --all --oneline --graph`             |


**流程图总结**：

```
Fork（如需） → Clone → 配置 Upstream → 切 main 并更新 
    → 创建 feature 分支 → 编码 → add → commit → push 
    → 创建 MR → Code Review → 修改代码 → push 
    → Approve → Merge to main → 拉取 main → 删除分支 → 开始下一轮
```