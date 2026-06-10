# GitLab 登录与 SSH 配置手册

## 一、修改记录

| 版本 | 内容 | 修改人 | 时间 |
|---|---|---|---|
| v1.0 | 初版 | 刘海阳 | 2026/3/5 |

## 二、登录 GitLab

### 2.1 打开登录页面
在浏览器中访问：

```text
http://192.168.101.173/
```

进入登录页面。

### 2.2 输入账号和密码
- 用户名：自己的姓名拼音
- 初始密码：`q123456.`

注意：密码末尾带英文句号 `.`

示例：
- 刘海阳 → `liuhaiyang`
- 肖川 → `xiaochuan`

### 2.3 首次登录后的处理
登录后系统会强制修改初始密码。

建议：
- 将初始密码修改为自己的常用密码
- 不必强行记住初始密码
- 如果忘记密码，可以联系管理员强制重置

### 2.4 修改页面语言
登录成功后，可以将页面语言切换为中文。

操作步骤：
1. 在页面中向下滚动
2. 找到语言设置位置
3. 选择中文
4. 点击保存
5. 刷新页面生效

## 三、添加 SSH 密钥

如果希望在 `push` / `pull` 仓库时不再频繁输入密码，可以配置 SSH Key。

### 3.1 安装 Git
如果本机尚未安装 Git，请下载安装：

[Git - Install for Windows](https://git-scm.com/install/windows)

### 3.2 设置 Git 用户名和邮箱
打开终端，执行：

```bash
git config --global user.name "你的名称，中英皆可"
git config --global user.email "你的邮箱"
```

说明：
- `user.name` 和 `user.email` 请改成你自己的信息
- 邮箱建议使用 `gitlab用户名@cks.com`
- Git 配置中的邮箱应尽量与 GitLab 账号邮箱保持一致

### 3.3 生成 SSH Key
在终端中执行：

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
```

### 3.4 选择保存位置
执行后会提示输入保存路径。默认文件通常为：

```text
C:\Users\<用户名>\.ssh\id_ed25519      # 私钥
C:\Users\<用户名>\.ssh\id_ed25519.pub  # 公钥
```

如果已有同名文件，建议换一个文件名。  
文件也可以生成在其他位置。

### 3.5 复制公钥
使用记事本或 VSCode 打开公钥文件，复制其中全部内容。

### 3.6 添加到 GitLab
进入 GitLab：

1. 打开用户设置
2. 进入 **SSH Keys**
3. 点击添加密钥
4. 将公钥粘贴到密钥栏
5. 标题填写一个便于识别的名称
6. （建议）删除到期时间，避免密钥过期后重新配置

## 四、常见问题

### 4.1 添加了 SSH Key 之后，push 仍然要求输入密码
可按以下顺序排查：

#### 4.1.1 步骤 1：测试 SSH 连接
在终端执行：

```bash
ssh -T git@192.168.101.173
```

如果仍然要求输入密码，按 `Ctrl + C` 取消。

#### 4.1.2 步骤 2：启动 ssh-agent
继续执行：

```bash
ssh-agent
```

如果出现如下错误：

```text
unable to start ssh-agent service, error :1058
```

说明服务被禁用。

#### 4.1.3 步骤 3：以管理员身份启用服务
按 `Win + X`，打开管理员终端，然后执行：

```powershell
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
```

#### 4.1.4 步骤 4：添加私钥
服务启动后执行：

```bash
ssh-add C:\Users\<用户名>\.ssh\<私钥名>
```

### 4.2 仍然无法免密访问
如果前面的方式仍然无效，可以检查 SSH 配置文件。

打开以下文件：

```text
C:\Users\<用户名>\.ssh\config
```

添加内容如下：

```sshconfig
Host gitlab
  HostName 192.168.101.173
  User git
  IdentityFile ~/.ssh/<私钥名>
```

保存后重新测试连接。

## 五、建议的使用方式
配置完成后，后续仓库地址建议优先使用 SSH 方式，这样可以减少每次操作时输入密码的次数。
