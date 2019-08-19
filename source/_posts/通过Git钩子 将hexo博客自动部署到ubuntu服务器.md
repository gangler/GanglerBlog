---
title: 通过Git钩子 将hexo博客自动部署到ubuntu服务器
date: 2019-08-10 19:25:00
author: Well Ding
img: /medias/featureimages/8.jpg
categories: hexo
tags:
  - git
  - node.js
  - ubuntu
---

为了博客能够在本地修改后，自动部署到服务器，折腾了好一整子，在此记录以免下次换服务器给忘了。

## 本地环境

首先需要基于hexo创建一个博客项目，具体见官网：https://hexo.io/zh-cn/
本地需要安装git,node.js,hexo

## 服务器环境

我用的是阿里云的轻量级服务器，还买了个域名，域名备案审核就等了几天，无语。。。
### 1. 配置服务器SSH
在阿里云的控制台---远程连接中，配置账号密码，然后本地就可以通过SSH远程连接服务器
我用的连接工具是Xshell
### 2. 安装环境
- **安装git**
```bash
sudo apt-get install git
```

- **安装node.js（通过nvm安装）**
```bash
wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh
source ~/.profile
nvm install stable
```

- **安装hexo**
```bash
npm install hexo-cli -g
npm install hexo-server -g
```

- **安装Nginx**
```bash
sudo apt-get update
sudo apt-get install git nginx -y
```

### 3.创建Git仓库
在/var/repo/下创建名为hexo_static的裸仓库。用如下命令
```bash
sudo mkdir /var/repo/
sudo chown -R $USER:$USER /var/repo/
sudo chmod -R 755 /var/repo/
```
```bash
cd /var/repo/
git init --bare hexo_static.git
```

### 4.配置Nginx托管文件目录
创建/var/www/hexo目录，用于Nginx托管，修改目录所有权和权限。
```bash
sudo mkdir -p /var/www/hexo
sudo chown -R $USER:$USER /var/www/hexo
sudo chmod -R 755 /var/www/hexo
```

随后修改Nginx的default设置，使root指向hexo目录。
```bash
sudo vim /etc/nginx/sites-available/default
```
注意一定要加sudo,否则会提醒default是只读文件.
修改文件中对应的项
```text
...

server {
listen 80 default_server;
listen [::]:80 default_server ipv6only=on;

root /var/www/hexo; # 需要修改的部分
index index.html index.htm;
...
```

重启Nginx服务，使得改动生效。
```bash
sudo service nginx restart
```

### 5.创建Git钩子
> 不清楚钩子是什么，可以看这里：https://aotu.io/notes/2017/04/10/githooks/index.html

在自动生成的 hooks 目录下创建一个新的钩子文件：
```bash
vim /var/repo/hexo_static.git/hooks/post-receive
```

在该文件中添加两行代码，指定 Git 的工作树（源代码）和 Git 目录（配置文件等）。
```text
#!/bin/bash
git --work-tree=/var/www/hexo --git-dir=/var/repo/hexo_static.git checkout -f
```

保存并退出文件，并让该文件变为可执行文件。
```bash
chmod +x /var/repo/hexo_static.git/hooks/post-receive
```

## 回到本地环境，继续配置
### 1.修改Hexo的默认配置
在站点config.yml中修改博客的地址url
```text
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'

url: http://server-ip # 没有绑定域名时填写服务器的实际 IP 地址。
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
```

### 2.通过Git部署
先在任意位置处打开powershell, 从服务器上把hexo_static仓库克隆下来, 以此来将服务器地址添加到受信任的站点中。
```bash
git clone root@server_ip:/var/repo/hexo_static.git
```

> 这里@前面的root就是ssh连接服务器时的用户

注意在第一次进行这一步时会提示是否继续，选yes即可。
再次编辑Hexo的config.yml文件，找到Deployment, 修改为
```text
deploy:
type: git
repo: root@server_ip:/var/repo/hexo_static.git
branch: master
```

最后记得安装Hexo部署到Git仓库的包。
```bash
npm install hexo-deployer-git --save
```
于是就可用 hexo d 命令来部署了。

以后本地文件修改后，只要运行如下命令，就可以同步到服务器了：
```bash
hexo clean
hexo g
hexo d
```


