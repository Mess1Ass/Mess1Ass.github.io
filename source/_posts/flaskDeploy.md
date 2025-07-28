---
title: Flask项目服务器部署
date: 2025-07-10 14:31:23
abstracts: 服务器部署
tags: [服务器部署]
cover: ./images/resume_icon.jpg
---

# 🚀 Flask 项目部署教程（使用 Gunicorn + Nginx + HTTPS）
本教程介绍如何在 Ubuntu 服务器上使用 Gunicorn + Nginx 部署 Flask 后端，并实现自定义域名绑定、HTTPS 自动签发及前端 Vercel 发布。

# 🧱 项目结构概览
toolBackend/           # Flask 后端项目
toolFrontend/          # React 前端项目（已托管 GitHub）
## ✅ 前提条件
一台 Ubuntu 服务器（推荐 20.04+）

已安装 Python3、pip、git

已绑定域名（如：48api.tool4me.cn）

已解析 A 记录至服务器公网 IP

## 📦 1. 安装依赖
``` bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```
推荐使用虚拟环境管理依赖：
``` bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```
## ⚙️ 2. 使用 Gunicorn 启动 Flask 项目
``` bash
gunicorn app:app --bind 127.0.0.1:5000 --workers 4 --daemon
app:app 表示：app.py 中的 app 实例
```

可将其写入 systemd 服务实现守护进程管理（可选）

## 🌐 3. 配置 Nginx 反向代理
编辑配置文件：
``` bash
sudo nano /etc/nginx/sites-available/your.domain.com
```
内容示例：
``` bash
server {
    listen 80;
    server_name your.domain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
创建软链接启用：
``` bash
sudo ln -s /etc/nginx/sites-available/your.domain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
## 🔒 4. 配置 HTTPS（Let's Encrypt）
使用 certbot 自动签发证书：
``` bash
sudo certbot --nginx -d your.domain.com
```
证书自动续期设置：
``` bash
sudo systemctl status certbot.timer  # 默认已启用自动续期
```
## 🌍 5. 前端部署到 Vercel
登录 vercel.com

绑定你的 GitHub 仓库（如：toolFrontend）

设置环境变量（如：API_URL 指向你的后端）

自动部署成功后绑定自定义域名（如：tool4me.cn）

## 🛡️ 6. 安全优化建议（可选）
配置 fail2ban 防爆破

使用 ufw 开启防火墙，仅开放 80 和 443 端口

定期备份 MongoDB 数据和日志文件

## 🎯 效果验证
后端 API 可通过 https://your.domain.com/api/xxx 访问

前端网页可通过 https://domain.com 访问

数据传输加密、全球 CDN 加速生效

## 👨‍💻 技术栈回顾
模块	技术栈
后端	Flask + Gunicorn + MongoDB
前端	React + Semi Design + Vercel
部署	Nginx + certbot + systemd
域名 & 安全	DNSPod + Let’s Encrypt
