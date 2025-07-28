---
title: 在服务器上搭建视频文件分发服务
date: 2025-07-10 14:31:23
abstracts: 服务搭建
tags: [服务搭建, ftp]
cover: ./images/resume_icon.jpg
---

# 在服务器上搭建视频文件分发服务（FTP + Nginx）
在开发前后端分离项目时，经常需要将静态资源如视频、文件等集中管理和访问。本篇记录如何在 Linux 服务器上创建一个 video 文件夹，配置 FTP 服务进行上传管理，并使用 Nginx 搭建文件下载站，实现前端直接访问链接播放视频的需求。

## 一、目标说明
✅ 在服务器上创建 /home/video 文件夹，用于存储视频资源

✅ 配置 FTP 服务（vsftpd），方便上传管理视频

✅ 使用 Nginx 映射该文件夹，实现网页可访问下载

✅ 前端使用直接链接 https://yourdomain.com/video/xxx.mp4 播放视频

## 二、创建视频文件夹
``` bash
sudo mkdir /home/video
sudo chown -R www-data:www-data /home/video
sudo chmod -R 755 /home/video
``` 
### ✅ 如果你想通过 FTP 上传并非 www-data 用户，也可以给对应用户授权（见下方 FTP 配置）。

## 三、配置 FTP 服务（vsftpd）
### 1. 安装 vsftpd
``` bash
sudo apt update
sudo apt install vsftpd -y
``` 
### 2. 添加 FTP 用户（用于上传）
``` bash
sudo adduser ftpuser
sudo passwd ftpuser
``` 
把 /home/video 设置为 ftpuser 的主目录：
``` bash
sudo usermod -d /home/video ftpuser
sudo chown -R ftpuser:ftpuser /home/video
``` 
### 3. 编辑 vsftpd 配置文件
``` bash
sudo nano /etc/vsftpd.conf
``` 
修改或添加以下内容：

``` conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30100
``` 
保存并重启服务：
``` bash
sudo systemctl restart vsftpd
``` 
### ✅ 你现在可以通过 FTP 工具（如 FileZilla）连接服务器上传视频文件到 /home/video 目录了。

## 四、配置 Nginx 静态文件下载站
### 1. 修改 Nginx 配置文件
假设你已经部署了 Nginx，打开你的网站配置：
``` bash
sudo nano /etc/nginx/sites-available/default
``` 
在 server 块中添加一段：

``` nginx
location /video/ {
    alias /home/video/;
    autoindex on;
    add_header Content-Disposition "inline";
    add_header Access-Control-Allow-Origin *;
}
``` 
### 2. 检查配置并重载 Nginx
``` bash
sudo nginx -t
sudo systemctl reload nginx
``` 
### 3. 访问测试
现在你可以通过以下地址直接访问或播放视频了：

``` arduino
https://yourdomain.com/video/demo.mp4
``` 
浏览器访问 /video/ 目录还可以列出文件清单（如果 autoindex 打开）。

## 五、前端链接修改
在 React 或 Vue 项目中，只需将视频资源链接改成：

``` tsx
<video src="https://yourdomain.com/video/sample.mp4" controls />
``` 
无需额外配置跨域，若接口为不同源，也可以通过 add_header Access-Control-Allow-Origin * 保证资源可访问。

## 六、其他建议
✅ 视频较多建议按日期或类别分文件夹上传，便于管理

✅ 可配合定时任务（cron）清理过期视频文件

✅ 如果你对权限有特殊需求，可单独创建用户组并控制文件访问

## 七、总结
通过本文配置，视频资源的上传、管理和前端访问将变得简单高效。适用于博客、后台管理系统、媒体内容平台等场景，结合 FTP 与 Nginx 的优势，即方便管理也支持高并发访问。

如有更多需求，例如用户上传、权限管理、下载鉴权等，也可以进一步集成 Flask 或 Django 等后端服务做二次开发。