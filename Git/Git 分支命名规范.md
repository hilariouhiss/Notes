# Git 分支命名规范

## 一、 核心前缀类型

| 前缀 | 用途 | 生命周期 | 源分支 | 目标分支 |
|------|------|----------|--------|----------|
| **`feature/`** | 新功能开发 | 随 MR 合并后删除 | `main` / `develop` | `main` / `develop` |
| **`bugfix/`** | 修复测试阶段发现的 Bug | 随 MR 合并后删除 | `develop` / `release` | `develop` / `release` |
| **`hotfix/`** | 生产环境紧急修复 | 随 MR 合并后删除 | `main` | `main` + `develop` |
| **`release/`** | 版本发布准备（冻结功能，只修 Bug） | 发布完成后删除 | `develop` | `main` |
| **`docs/`** | 仅文档更新 | 随 MR 合并后删除 | `main` / `develop` | `main` / `develop` |
| **`refactor/`** | 代码重构（无功能变更） | 随 MR 合并后删除 | `develop` | `develop` |
| **`test/`** | 补充测试用例 | 随 MR 合并后删除 | `develop` | `develop` |
| **`chore/`** | 构建脚本、依赖升级等杂项 | 随 MR 合并后删除 | `develop` | `develop` |
| **`experiment/`** | 技术预研/POC（可能不合并） | 手动清理 | 任意 | 不合并 |

## 二、 命名格式详解

### 2.1 基础格式

```
<prefix>/<description>
<prefix>/<issue-id>-<<description>
<prefix>/<author>-<<issue-id>-<<description>
```
### 2.2 各段含义

| 段 | 说明 | 示例 |
|----|------|------|
| **`<prefix>`** | 分支类型，必须小写 | `feature`, `bugfix`, `hotfix` |
| **`<issue-id>`** | 关联的 GitLab Issue / Jira 工单号 | `PROJ-123`, `#456` |
| **`<description>`** | 简短描述，动词开头，说明做了什么 | `add-login-api`, `fix-memory-leak` |
| **`<author>`** | 多人协作时标识开发者（可选） | `zhangsan`, `lhy` |

## 三、 书写约定

### 3.1 分隔符
- **必须使用**连字符 `-` 作为单词分隔符
- **禁止使用**下划线 `_` 或驼峰命名（部分 CI/CD 脚本和 URL 对下划线支持不佳）

**✅ 正确**
```
feature/add-user-auth
bugfix/PROJ-456-fix-null-pointer
hotfix/789-memory-leak-in-cache
```

**❌ 错误**
```
feature/add_user_auth          # 含下划线
feature/AddUserAuth            # 驼峰命名
feature/PROJ 456 fix leak      # 含空格
feature/adduserauth            # 无分隔符，不可读
```

### 3.2 长度控制
- 总长度建议 **不超过 50 个字符**
- 描述部分控制在 **3~5 个单词**

### 3.3 语义规范
- 使用**动词原形**开头（`add`, `fix`, `update`, `remove`, `refactor`）
- 避免无意义词汇（`temp`, `test`, `xxx`, `final`, `final2`）

## 四、 完整示例

### 4.1 功能开发
```bash
# 基础版
git checkout -b feature/add-jwt-authentication

# 关联 Issue 版（推荐）
git checkout -b feature/142-add-jwt-authentication

# 多人协作版（大型团队可选）
git checkout -b feature/lhy-142-add-jwt-authentication
```

### 4.2 Bug 修复
```bash
# 开发阶段发现的 Bug
git checkout -b bugfix/215-fix-login-redirect-loop

# 生产热修复（需同时打 Tag）
git checkout -b hotfix/318-fix-payment-callback-timeout
```

### 4.3 版本发布
```bash
# 准备发布 v2.3.0
git checkout -b release/v2.3.0

# 发布分支上的 Bug 修复
git checkout -b bugfix/v2.3.0-fix-api-docs
```

## 五、 与 GitLab Issue 联动（最佳实践）

### 5.1 自动关联
若分支名包含 Issue ID，GitLab 可配置为**自动创建分支-Issue 关联**：

```bash
# GitLab 原生格式（#号在 shell 中需转义，建议省略）
feature/123-add-oauth-login
```

**效果**：
- MR 合并后，Issue `#123` 自动关闭（需在 MR 描述中写 `Closes #123`）
- Issue 页面自动显示关联的分支和 MR

## 六、 进阶：多环境/多版本管理

对于需要维护多个版本线的项目（如同时维护 v2.x 和 v3.x）：

```bash
# 在版本分支上开修复
git checkout -b bugfix/2.x-fix-security-patch   # 基于 release/v2.x
git checkout -b bugfix/3.x-fix-security-patch   # 基于 release/v3.x
```

## 七、 速查清单：创建分支前自检

- [ ] 前缀是否来自标准列表（`feature/`, `bugfix/`, `hotfix/` 等）
- [ ] 是否基于正确的源分支（`main`/`develop`/`release`）
- [ ] 是否包含关联的 Issue ID（如有）
- [ ] 描述是否用动词开头、小写、连字符分隔
- [ ] 总长度是否 ≤ 50 字符
- [ ] 是否不含大写、下划线、空格、特殊符号

## 八、 团队规范模板

```markdown
## 分支命名规范

格式：`<type>/<<issue-id>>-<description>`

### 类型前缀
- `feature/`：新功能
- `bugfix/`：Bug 修复
- `hotfix/`：线上紧急修复
- `release/`：版本发布
- `docs/`：文档更新
- `refactor/`：代码重构

### 示例
- `feature/156-add-wechat-login`
- `bugfix/201-fix-race-condition-in-cache`
- `hotfix/urgent-fix-payment-signature`

### 禁止
- 个人姓名前缀（除非团队明确要求）
- `temp/`, `test/`, `wip/` 等无明确语义前缀
- 使用 `_` 或驼峰命名
```