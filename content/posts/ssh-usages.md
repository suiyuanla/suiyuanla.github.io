---
title: "SSH使用记录"
date: "2024-09-19T14:14:57+08:00"
draft: true
categories:
  - Development
  - linux
tags:
  - archlinux
  - coding
series:
  - "arch从零开始"
---

日常中使用经常连接服务器，有的时候还需要使用跳板机和端口转发，于是简单记录一下自己使用 ssh 的经验。

## 基础配置

ssh 的默认配置文件在 Home 目录下的 `.ssh` 文件夹中，使用`ssh-keygen -C "<描述内容>"`会在这个文件夹中生成`id_rsa`和`id_rsa.pub`文件。如果是复制过来的密钥和公钥文件，需要确保其权限为仅当前用户可读可写(`chmod 600`)，如果其他用户有读写权限则使用 ssh 连接服务器时会报错。将公钥复制到要连接的服务器的指定用户的`~/.ssh`文件夹下的`authorized_keys`文件中，就可以免密登录。

如果需要关闭服务器的密码登录权限，则需要在`/etc/ssh/sshd_config`中设置`PasswordAuthentication no`禁用密码登录。

> 注意：你应该先把公钥复制到 authorized_keys 中，并且测试过了可以免密登录，再禁用密码登录。

## 命令行使用

常用的命令行命令有：

```bash
# 连接到指定服务器，username是登录用户名，
# hostname可以是域名和IP，可以使用-p指定端口，默认为22
ssh username@hostname
# 在本机指定端口开启代理，相当于一个socks代理
ssh -TND <端口> username@hostname

# 发送到指定端口的数据都会经过hostname
# 所在的服务器转发到指定远程主机的对应端口
ssh -TNL <本地端口>:<远程主机ip>:<远程端口> username@hostname
# 例子，发送到本地8001端口的数据都会经过10.10.10.10转发，发送到10.10.10.10所在网络环境下的192.168.0.3:80
ssh -TNL 8001:192.168.0.3:8006 username@10.10.10.10

# 使用跳板机
# 假设你在host1 ... hostn都配置了公钥免密登录，你可以经过host1 跳转到 host2 ... 链式跳转连接到 hostn
ssh -J user1@host1 user2@host2 ... usern@hostn
```

## 使用 ssh 配置文件

配置文件是`.ssh`文件夹内的`config`文件，下面是一个例子：

```bash
# 在命令行使用ssh jump相当于
# 使用ssh user1@10.72.0.1
Host jump
  Hostname 10.72.0.1
  User user1

# 在命令行使用ssh host 相当于
# 使用ssh -J jump user2@192.168.0.35
# 相当于 ssh -J user1@10.72.0.1 user2@192.168.0.35
Host host
  Hostname 192.168.0.35
  User user2
  ProxyJump jump
```

配置文件还有很多其他命令选项，用到的时候再查~~。

## 配置 ssh 代理服务开机启动

平时对我来说常见的一个情况是：

需要使用跳板机访问内网的网页服务，如果使用 `-TNL` 的话过于繁琐，对内网每个网页，都要进行端口映射，所以应该使用`-TND`直接在本地端口开启一个 ssh 代理。

浏览器可以配合[这个插件](https://github.com/FelisCatus/SwitchyOmega)，让指定网段的域名走 ssh 端口的代理。

这么一来代理的问题解决了，但是每次开机都要打开个终端运行这个命令太麻烦了，而且一不小心就把它关闭了。

在**Aria2 全能下载器配置**这个文章的编写过程中，我学会了使用 systemd-service 来运行自定义服务，因此就编写了如下一个 service：

```ini
# ~/.config/systemd/user/ssh-proxy@.service
[Unit]
Description=SSH Tunnel to %i
Wants=network-online.target
After=network-online.target nss-lookup.target

[Service]
# 替换8001为自定义端口
ExecStart=/usr/bin/ssh -TND 8001 %i
Restart=on-failure

[Install]
WantedBy=default.target
```

随后运行下面的命令：

```bash
systemctl --user daemon-reload
# <host>是在~/.ssh/config 中写好的服务器，可以直接免密登录
# @后面的<host>会作为参数替换.service中的%i
systemctl --user enable --now ssh-proxy@<host>.service
# 查看服务的状态
systemctl --user status ssh-proxy@<host>.service
```

这样配置好后，就可以在登录后愉快的使用 ssh 代理了:)。
