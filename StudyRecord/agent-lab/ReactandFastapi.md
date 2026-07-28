data: 2026/7/27

## 一些linux操作
git bash 和 cmd 的区别  
git bash 为模拟linux 风格，cmd为windows风格。

1. pwd 定位； ls 列出文件
2. cd ，到哪个子文件去; cd .. 回到上级目录
3. touch 创建文件
4. cat 粘贴内容
5. start用默认程序（浏览器）打开文件

## 不同电脑如何通过浏览器分享内容
打开网站实际上就是访问远程服务器提供的内容。
这就涉及到计算机网络的知识，详见 [计算机网络笔记](../BasicLearn/net.md)
1. 你在浏览器里输入一个域名
2. 浏览器先去查这个域名对应的 IP 地址
3. DNS 把这个域名对应的 IP 地址告诉浏览器
4. 浏览器根据 IP 地址找到那台服务器
5. 浏览器再通过某个端口访问这台服务器上的具体服务
6. 服务器把网页内容返回给浏览器
7. 浏览器把这些内容显示成你看到的网页

## 租赁服务器

自己电脑连接的是流量、wifi，本质上是路由器分配的内网地址，可以主动发连接，但是不能被主动找到，只有公网IP可以。

外面的人 → 22端口 → SSH 登录你的服务器
外面的人 → 80端口 → 浏览器看你的网站  
外面的人 → ping   → 检测你的服务器是否活着

其他所有端口（3306、443、3000等）默认都是拒绝的，外面连不上。

限制22端口不向全IP开放。

### SSH 密钥登录配置（禁用密码登录）

**第一步：本地生成密钥对**
在 CMD 中执行：
```cmd
ssh-keygen -t ed25519 -C "备注"
```
一路回车，默认保存在 `C:\Users\你的用户名\.ssh\`，生成两个文件：
- `id_ed25519` —— 私钥（自己保管，绝不外传）
- `id_ed25519.pub` —— 公钥（放到服务器上）

**第二步：上传公钥到服务器（CMD）**
```cmd
type %USERPROFILE%\.ssh\id_ed25519.pub | ssh ubuntu@你的IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
输入一次密码，公钥就写入服务器了。

> 注意：Git Bash 里改用 `cat /c/Users/...`路径；PowerShell 里改用 `$env:USERPROFILE`。

**第三步：测试密钥登录**
```bash
ssh ubuntu@你的IP
```
不用输密码直接进去 → 成功。

**第四步：禁用密码登录**
确认密钥能用后，在服务器上执行：
```bash
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```
> ⚠️ 不要关掉当前窗口！新开一个终端确认密钥能登录再关旧的。

配完之后：没有你的私钥，谁都进不来。22端口对全世界开放也安全。

### root 和 ubuntu
- `root`：超级管理员（最高权限），腾讯云 Ubuntu 镜像默认禁用 root 登录
- `ubuntu`：普通用户，也是你服务器的用户名，加 `sudo` 就能提权做管理操作

ssh ubuntu@101.33.225.107

## Nginx
软件，监听80端口，返回网页内容。
安装：
sudo apt update
sudo apt install nginx -y
确认 Nginx 已经在运行：
systemctl status nginx

sudo vim /var/www/html/index.nginx-debian.html
编辑模式
i 编辑；esc退出；：wq 保存

## 前端基础HTML


