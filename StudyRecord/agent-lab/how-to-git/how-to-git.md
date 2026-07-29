# Git 与 GitHub 连接流程

## 1. 前置准备

| 步骤 | 命令 / 操作 | 说明 |
|------|-------------|------|
| 安装 Git | `git --version` | 确认已安装 |
| 配置身份 | `git config --global user.name "你的名字"` `git config --global user.email "你的邮箱"` | commit 时会附上这个身份 |
| GitHub 账号 | 注册 github.com 账号 | — |

## 2. 认证（让 GitHub 认识你）

三种方式，选一种即可：

| 方式 | 适用场景 | 要点 |
|------|---------|------|
| **Personal Access Token**（推荐） | HTTPS 推送 | 在 [Settings → Developer settings → Tokens](https://github.com/settings/tokens) 生成，推送时输入 token 作为密码 |
| **SSH Key** | 免密推送 | `ssh-keygen` 生成密钥对，公钥上传到 GitHub Settings → SSH Keys |
| **GitHub CLI** (`gh`) | 命令行一体化 | `gh auth login` 浏览器授权 |

当前使用的是 **HTTPS + Token** 方式。

## 3. 连接 GitHub 的四种典型场景

### 场景 A：推自己的新项目

```bash
# 本地已有代码
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/你的用户名/仓库名.git
git push -u origin main
```

### 场景 B：克隆别人的项目

```bash
git clone https://github.com/作者/仓库名.git
cd 仓库名
# 改了代码之后…
git add .
git commit -m "描述你的改动"
git push
```

### 场景 C：Fork → 改 → PR（参与开源）

```bash
# 1. GitHub 网页上点 Fork，复制到你自己的账号下
# 2. clone 你自己 fork 的副本
git clone https://github.com/你的用户名/仓库名.git

# 3. 添加原仓库为 upstream（追踪原项目更新）
git remote add upstream https://github.com/原作者/仓库名.git

# 4. 改代码 → commit → push 到自己的 fork
git add .
git commit -m "修复了某个 bug"
git push origin main

# 5. 去 GitHub 网页发起 Pull Request
#    从 "你的用户名/仓库名:main" → "原作者/仓库名:main"
```

### 场景 D：远程有更新，同步到本地

```bash
git pull origin main        # 拉取 + 合并（简单场景）
# 或者
git fetch origin            # 先看看有什么更新
git rebase origin/main      # 把自己的 commit 接在最新后面（推荐）
```

## 4. 核心概念速查

```
工作区 ──git add──▶ 暂存区 ──git commit──▶ 本地仓库 ──git push──▶ GitHub
  │                    │                      │
  │ ◀─────────────────────────────────── git pull ─────────────────│
  │                    │                      │
  └── 你编辑的文件      └── 准备提交的改动     └── .git 目录
```

| 命令 | 作用 |
|------|------|
| `git remote -v` | 查看当前连接的远程仓库地址 |
| `git remote add 别名 URL` | 添加远程仓库（origin、upstream 是约定俗成的别名） |
| `git push -u origin main` | 推送并建立追踪关系（之后直接 `git push` 即可） |
| `git pull` | 拉取远程更新并合并 |
| `git clone URL` | 下载整个仓库到本地 |

## 5. 当前配置

```
远程仓库: https://github.com/3361880969LXY/agent_lab
SSL 后端: openssl（schannel 与代理不兼容）
代理:     127.0.0.1:7897
认证:     HTTPS + Personal Access Token
```

## 6. 远程服务器连接 GitHub

> 以 `ssh ubuntu@101.33.225.107` 登录的云服务器为例

### 步骤

```bash
# 1. 登录服务器
ssh ubuntu@101.33.225.107

# 2. 服务器上安装 Git（通常已预装）
sudo apt update && sudo apt install git -y

# 3. 配置身份
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 4. 生成 SSH 密钥（一路回车，不设密码）
ssh-keygen -t ed25519 -C "你的邮箱"

# 5. 查看公钥，复制全部内容
cat ~/.ssh/id_ed25519.pub
# 输出类似: ssh-ed25519 AAAAC3... 你的邮箱
```

### 添加公钥到 GitHub

1. 复制终端输出的公钥
2. 打开 GitHub → Settings → [SSH and GPG keys](https://github.com/settings/keys)
3. 点击 **New SSH key**
4. Title 填服务器标识（如 `腾讯云服务器`），Key 粘贴公钥
5. 点击 **Add SSH key**

### 验证连接

```bash
# 服务器上运行
ssh -T git@github.com

# 首次连接会提示：
#   Are you sure you want to continue connecting (yes/no/[fingerprint])?
# 输入 yes 回车

# 成功后显示：
#   Hi 你的用户名! You've successfully authenticated...
```

### 服务器上的 Git 操作

```bash
# clone 项目到服务器
git clone git@github.com:你的用户名/仓库名.git

# 或者把已有项目改为 SSH 推送
git remote set-url origin git@github.com:你的用户名/仓库名.git

# 正常 push / pull
git push origin main
git pull origin main
```

> **提示**：SSH 方式比 HTTPS + Token 更适合服务器，因为不需要每次输入密码，也不存在 token 泄露到代码里的风险。

## 7. 安全提醒

- **永远不要把 token 写进代码或 notebook**，用环境变量代替：

  ```python
  import os
  TOKEN = os.environ.get("GITHUB_TOKEN")  # 从环境变量读取
  ```

- 如果 token 不慎泄露，立刻去 GitHub 撤销旧 token、生成新的。
- `.gitignore` 中排除敏感文件（`.env`、token 文件等）。
