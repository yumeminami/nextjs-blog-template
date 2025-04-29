---
title: "How to setup VNC server"
date: "2025-04-29"
draft: false
keywords: ["linux", "vnc", "system service"]
summary: ""
featured: true
---


## ✅ 1. 安装 VNC 服务器 和 桌面环境

```bash
sudo apt update
sudo apt install tigervnc-standalone-server xfce4 xfce4-goodies -y
```

## ✅ 2. 设置 VNC 密码

```bash
vncpasswd
```

## ✅ 3. 创建 xstartup 文件（桌面环境）

```bash
vim ~/.vnc/xstartup
```

`xstartup` content

```bash
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
#exec startxfce4
dbus-launch startxfce4
```

add execute right:

```bash
chmod 777 ~/.vnc/xstartup
```

## ✅ 4. 测试运行 VNC

```bash
vncserver :2 -localhost no

vncserver -kill :2 # kill the vnc server 
```

## ✅ 5. 配置 systemd 开机自启动（推荐）

**⚠️ 我曾经被 AI，论坛的各种模版折磨得死去活来**

AI 或者各种博客论坛 的 `vncserver.service`

```bash
[Unit]
Description=Start TigerVNC server at startup
After=syslog.target network.target

[Service]
Type=forking
User=%i
PAMName=login
PIDFile=/home/%i/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver :%i -localhost no
ExecStop=/usr/bin/vncserver -kill :%i

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

启动 service

```bash
sudo systemctl daemon-reload
sudo systemctl restart vncserver@1
Job for vncserver@2.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status vncserver@2.service" and "journalctl -xe" for details.
```

后来我修改为 `vncserver.service`

```bash
[Unit]
Description=Remote desktop (VNC)
After=syslog.target network.target

[Service]
Type=forking
User=rongman
Group=rongman

PIDFile=/home/rongman/.vnc/ubuntu:1.pid
ExecStart=vncserver -geometry 1920x1080 :1 -localhost no
ExecStop=/usr/bin/vncserver -kill :1

[Install]
WantedBy=multi-user.target
```

区别在于我显示编码了占位符的内容
