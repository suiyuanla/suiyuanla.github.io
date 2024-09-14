---
title: "Aria2全能下载器配置"
date: "2024-09-14T14:11:14+08:00"
draft: true
categories:
  - fun
tags:
  - archlinux
series:
  - "没什么用的记录"
---

## 简介

本篇记录一下自己使用的多线程下载器方案，由于我日常使用的操作系统是 Archlinux，因此是 linux 下的配置，windows 下使用需要进行修改，并且可能有其他更好的方案。

[Aria2](https://github.com/aria2/aria2)是一个多线程下载器，它支持 HTTP(s)，FTP，STP，BitTorrent 等协议的下载，基本上安装它一个就足以应付日常的所有下载需求了，这也是我选择它的原因。由于我没有离线下载的需求，因此本篇文章并不会涉及相关内容，本篇更关注如何使日常使用 aria2 更舒适方便。

## 安装

在 Archlinux 上安装 aria2:

```bash
paru -S aria2
```

安装好后，aria2 需要进行一些配置才适合使用，我使用的是["Aria2 完美配置"](https://github.com/P3TERX/aria2.conf)这个项目。

```bash
# 将其clone到~/.config/aria2
git clone https://github.com/P3TERX/aria2.conf.git ~/.aria2
```

克隆下来的 aria2.conf 需要进行一些修改，按自己需要修改，需要注意的是，aira2 的配置文件不支持`~`为 Home 目录，需要将下面示例的`~`替换为绝对路径：

```ini
# 下载目录。可使用绝对路径或相对路径, 默认: 当前启动位置
dir=~/Downloads
# 从会话文件中读取下载任务
input-file=~/.aria2/aria2.session
# 会话文件保存路径
# Aria2 退出时或指定的时间间隔会保存`错误/未完成`的下载任务到会话文件
save-session=~/.aria2/aria2.session

# IPv4 DHT 文件路径，默认：$HOME/.aria2/dht.dat
dht-file-path=~/.aria2/dht.dat
# IPv6 DHT 文件路径，默认：$HOME/.aria2/dht6.dat
dht-file-path6=~/.aria2/dht6.dat

# 下载停止后执行的命令
# 从 正在下载 到 删除、错误、完成 时触发。暂停被标记为未开始下载，故与此项无关。
on-download-stop=~/.aria2/delete.sh
# 下载完成后执行的命令
# 此项未定义则执行 下载停止后执行的命令 (on-download-stop)
on-download-complete=~/.aria2/clean.sh

# RPC 监听端口, 默认:6800
rpc-listen-port=6800
# RPC 密钥 改为自己的密钥
rpc-secret=P3TERX
```

修改好配置以后，可以直接命令行尝试使用 aria2c，如果能正常使用，则配置正确。由于我是在笔记本电脑安装 arch，因此会有开关机的需求，需要编写一个 systemd 的 service 确保 aria2 每次开机都能自动启动：

```ini
# 文件位置：~/.config/systemd/user/aria2cd.service
[Unit]
Description=aria2 Daemon

[Service]
Type=simple
ExecStart=/usr/bin/aria2c --conf-path=/path/to/conf

[Install]
WantedBy=default.target
```

除了这些配置，可以手动运行`~/.aria2/tracker.sh ~/.aria2/aria2.conf`，这个脚本将更新 aria2.conf 里的 BT Tracker 列表，可以写一个定时任务让其定期更新 Tracker:

```ini
# 首先编写一个更新tracker的service
# 文件位置：~/.config/systemd/user/aria2-update-tracker.service
[Unit]
Description=Update aria2 tracker
After=network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c "/path/to/home/.aria2/tracker.sh /path/to/home/.aria2/aria2.conf"

[Install]
WantedBy=default.target

# 再编写一个定时任务
# 文件位置：~/.config/systemd/user/aria2-update-tracker.timer
[Unit]
Description=Run aria2-update-tracker.service every 12 hours

[Timer]
# 在开机30s后运行第一次
OnBootSec=30s
# 每12h运行一次，如果只想开机自动更新可以注释这一行
OnActiveSec=12h
Persistent=true

[Install]
WantedBy=default.target
```

启动对应的服务（不需要 sudo）：

```bash
systemctl --user daemon-reload
systemctl --user enable --now aria2cd.service
systemctl --user enable --now aria2-update-tracker.timer

# 使用下列命令查看service状态
systemctl --user status aria2cd.service
systemctl --user status aria2-update-tracker.timer
```

## 浏览器配置

安装好了，就得来设置如何使用了，这里推荐一个扩展[**download_with_aria2**](https://github.com/jc3213/download_with_aria2)，可以在 Edge 和 Firefox 上安装使用。

在扩展设置页面配置好 json-rpc 服务器和密钥，默认为`http://localhost:6800/jsonrpc`，端口和密钥在前面的 aria2.conf 里配置相同。然后在**扩展设置**->**捕获浏览器下载**中勾选 **启用捕获功能**，并且在下面的 **监视域名**中添加匹配规则 **\*** 就可以正常捕获浏览器的下载请求了。

进一步，可以使用[油猴插件 BETA 版](https://www.tampermonkey.net/)，安装[(改)网盘直链下载助手](https://github.com/hmjz100/Online-disk-direct-link-download-assistant)，随便打开一个百度网盘分享连接，可以看到脚本的设置入口，在设置里先点亮，随后查看具体设置，确保端口和密钥都正确无误，随后将他人分享的文件保存到自己网盘，选择 rpc 下载，即可发送到 aria2 进行下载。
