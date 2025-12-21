---
title: 从零开始部署Hexo到服务器
date: 2025-12-20 01:00:00
tags: 
  - 部署
categories: 
  - 部署
description: 闲出屁来的产物。
---


# 从零开始部署Hexo到服务器

## 目录

- [0 运行环境](#0-运行环境)

- [1 整体流程](#1-整体流程)

- [2 本地安装](#2-本地安装)

- [3 服务器安装](#3-服务器安装)
  
  - [3.1 安装Node.js和Git](#31-安装nodejs和git)
  
  - [3.2 安装和配置Nginx](#32-安装和配置nginx)
  
  - [3.3 Git配置](#33-git配置)
- [4 部署Hexo到服务器](#4-部署hexo到服务器)
  
    

## 0 运行环境

本地Windows11

服务器Debian 12

## 1 整体流程

1. 本地运行hexo deploy，通过git推送到服务器
2. 在git仓库配置钩子脚本，在接收到推送后自动将最新的静态文件同步覆盖到指定的工作目录
3. 配置nginx服务器，将请求指向该工作目录

## 2 本地安装

### 2.1 安装 Node.js 和 Git

[Node.js安装链接](https://nodejs.org/dist/v24.12.0/node-v24.12.0-x64.msi)

[Git安装包链接](https://git-scm.cn/downloads/win)

#### 使用`git -v`,`node -v`验证安装

```
C:\Users\Admin>git -v
git version 2.51.0.windows.1
```

```
C:\Users\Admin>node -v
v25.2.0
```

### 2.2 安装hexo

`npm install -g hexo-cli` 

### 2.3 初始化hexo

新建一个文件夹 ，并在该文件夹路径下的终端运行`hexo init`，并安装hexo-deployer-git 插件`npm install hexo-deployer-git --save`

## 3 服务器安装

### 3.1 安装Node.js和Git

#### 更新系统

```
sudo apt update
sudo apt upgrade -y
```

#### 安装

```
sudo apt install -y curl
# 安装最新Node.js
curl -fsSL https://deb.nodesource.com/setup_current.x | sudo -E bash -
sudo apt install nodejs -y
# 安装Git
sudo apt install -y git
```

#### 验证安装

```
node -v
npm -v
git -v
```

### 3.2 安装和配置Nginx

#### 安装Nginx

```
apt install -y nginx
```

##### 配置Nginx

创建Hexo运行目录`mkdir -p /data/hexo` 进入到`/etc/nginx/conf.d`目录下，通过`vim default.conf`创建配置文件

- 无域名配置文件 (修改IP 为实际IP值)

  ```
  server {
      listen        80;
      listen   [::]:80;
      server_name  IP;
  
      location / {
          root   /data/hexo;
          index  index.html index.htm;
      }
  
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  }
  ```

- 有域名且有SSL证书(修改`server_name`和SSL证书实际存放位置`ssl_certificate`,`ssl_certificate_key`)

```
server {
    listen 80;
    listen [::]:80;
    server_name 你的域名;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     yourdomain.com.pem;
    ssl_certificate_key yourdomain.com.key;

    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;
    ssl_session_tickets off;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    root /data/hexo;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

- 使用反向代理

  （待更新）

#### 3.3 Git配置
##### 3.3.1 创建Git用户
`useradd git`
<br>
修改git用户权限
```
sudo chmod 740 /etc/sudoers
sudo vim /etc/sudoers
```
添加`git ALL=(ALL) ALL` 
```
# User privilege specification
root    ALL=(ALL:ALL) ALL
git     ALL=(ALL:ALL) ALL
```
继续修改权限并添加密码
```
sudo chmod 400 /etc/sudoers
sudo passwd git
```

##### 3.3.2 ssh免密登录

在本机打开Git终端，创建密钥对`ssh-keygen`

使用`ssh-copy-id`工具将公钥上传到服务器`

```
ssh-copy-id -i ~/.ssh/id_ed25519.pub username@server_ip
```

**如果没有改命令(手动配置)**

1. 复制本地公钥内容

2. 登录到服务器，切换到git用户并执行

   ```
   mkdir -p ~/.ssh
   echo "粘贴公钥内容" >> ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```
3. 测试免密登录  `ssh -v git@YourServerIp`

#### 3.3.3 创建Git目录，配置hooks文件
```
cd ~
git init --bare hexo.git
vim ~/hexo.git/hooks/post-receive
```
插入`git --work-tree=/data/hexo --git-dir=/home/git/hexo.git checkout -f`

**修改hooks文件的权限**
```
chmod +x ~/hexo.git/hooks/post-receive
cd ~
sudo chmod -R 777 /data/hexo
```
**重启服务器**
## 4 部署Hexo到服务器
(1) 编辑本地hexo项目中`_config.yml`文件
```
deploy:
  type: git
  repo: git@YourServerIp:/home/git/hexo.git
  branch: master
```
(2) 配置Git全局变量
<br>打开本机终端 
```
git config --global user.name "xxx"
git config --global user.email "xxx@xx.com"
```
(4) 生成静态文件并推送到服务器
`hexo clean&hexo deploy`

**至此，已经全部配置完毕了**