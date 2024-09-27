---
title: "linux下使用雷神加速器：基于docker"
date: "2024-09-27T09:10:55+08:00"
draft: false
categories:
  - linux
  - fun
tags:
  - archlinux
series:
  - "没什么用的记录"
  - "arch从零开始"
---

## 简介

自从我全面迁移到 archlinux，目前手里已经没有 windows 的设备。在日常使用中，我有遇到了两个痛点：

1. 腾讯会议共享屏幕的方案不够完美
2. 之前在未接触 linux 时，充值了 2600h 的游戏加速时长，目前还有 1900h，在 linux 下无法使用

对于前者，除非腾讯重构腾讯会议，不然没有好的办法，而且工作时一般没有权力来决定不用腾讯会议（。而对于后者，我在网上搜寻了许多资料，一种可行的方案是跑一个 windows 虚拟机，在里面运行游戏加速器，然后代理主机的流量。但是 windiows 虚拟机还是太重了，我想寻求一种更轻量的方案。

## 探索

我在网上进行了大量的检索，先是发现雷神加速器有 openwrt 插件，并且有网友使用的是 openwrt 虚拟机桥接到局域网来代理流量，可以正常的工作。其实到这里也足够了，openwrt 虚拟机相比 windows 虚拟机，占用资源已经很小了。但是本着折腾的原则，我想在 docker 里跑 openwrt 来代理本机的流量。

如果不想折腾，你应该：

1. 首先尝试购买一个软路由，在里面运行雷神加速器插件。
2. 安装一个 openwrt 虚拟机，桥接网络的情况下代理本机。
3. 最后再尝试使用 docker 运行 openwrt 来跑雷神加速器。

关于这个方案的优缺点，优点是省钱，缺点是折腾且不适用于笔记本电脑。因为 docker 里要配置静态 ip。个人感觉这个方案更适合不来回搬动的台式机。

在跟着本记录做之前，你应该：

1. 对 docker 的使用有一定的了解
2. 对局域网、Nat、桥接有一定的了解
3. 对 openwrt 有一定了解

（虽然现学也行

## 安装

首先是找到一个 openwrt 镜像，我个人懒得自己做，用的是这个[openwrt-x86](https://hub.docker.com/r/piaoyizy/openwrt-x86)。

里面有如何使用 docker 安装的教程，我详细说明一下：

```bash
# 拉取镜像
docker pull piaoyizy/openwrt-x86:latest
# 此处的eth0应该为你使用的网卡，可以用ip a查看
sudo ip link set eth0 promisc on
# 此处创建一个macvlan的网络，其子网是当前物理机
# 局域网所在的网络，如“192.168.0.0/24”
# 网关也是物理机的网关，如“192.168.0.1“
# 最后一个参数macnet是创建的docker网络的名字，可以自定义
docker network create -d macvlan --subnet=10.10.10.0/24 --gateway=10.10.10.1 -o parent=eth0 macnet
# 运行指定的docekr容器，此处要特权运行，ip指定为所在局域网里一个固定的IP（不要这个选项应该也可以？）
docker run -d --name=OpenWrt --restart always --privileged --network macnet --ip 10.10.10.30 piaoyizy/openwrt-x86:latest
```

在运行后，可以使用`docker ps`查看容器是否真的在运行，随后使用`docker exec -it OpenWrt /bin/bash`来进入到容器内部。在容器内部运行下面的命令:

```bash
vi /etc/config/network
# 将下列内容中的ipaddr进行修改
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '10.10.10.6'
        option netmask '255.255.255.0'
        option ip6assign '60'
# 修改后
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '<静态ip>'
        option gateway '<所在网络网关>'
        option netmask '<所在网络对应的掩码>'
        option ip6assign '60'

# 在修改完后，重启网络
/etc/init.d/network restart
# 查看修改是否生效
ip a
```

其实此时，openwrt已经配好了，可以用手机或其他设备，可以在浏览器访问到openwrt的ip，但是由于内核限制，宿主机无法访问到macvlan的容器。

最简单的解决方案是双网卡，即电脑同时有两个网卡，那么使用另一个网卡连接到网络，就可以访问到容器。比如我的电脑同时有有线和无线网卡，我基于有线网卡做了macvlan网络，给容器使用，虽然本机也可以使用有线网卡上网，但是无法访问到容器。如果此时我在networkmanager断开（kde图形界面）自己的有线网，并不会影响macvlan 容器的网络。随后本机使用无线网连接到同一个网络，就可以访问到容器。

另一种解决的方案则是再在原网卡开一个macvlan虚拟网卡，利用这个网卡，宿主机就可以与容器通信。由于networkmanager管理macvlan虚拟化的网卡有问题，因此我使用的是systemd-networkd来管理，其配套的systemd-resolved服务也应该打开。需要在`/etc/systemd/network`下创建下列几个文件：

```ini
# 假设eth0是你真实的网卡
# /etc/systemd/network/eth0.network
[Match]
Name=eth0
     
[Link]
RequiredForOnline=carrier

[Network]
MACVLAN=macproxy
DHCP=no
IPv6AcceptRA=false
LinkLocalAddressing=no
MulticastDNS=false
LLMNR=false
# 假设macproxy是你想要的macvlan网卡的名字
# /etc/systemd/network/macproxy.netdev
[NetDev]
Name=macproxy
Kind=macvlan
# 参考格式 MACAddress=00:00:00:00:00:00
MACAddress=<自定义一个固定的mac地址>

[MACVLAN]
Mode=bridge

# /etc/systemd/network/macproxy.network
[Match]
Name=macproxy

[Link]
RequiredForOnline=routable

[Network]
BindCarrier=eth0
# Address为你想要的固定ip，DHCP或许也可以
Address=192.168.0.31/24
# Gateway和dns应该是openwrt容器的ip，这样通过新的macvlan网络的流量都会被容器代理
Gateway=192.168.0.30
DNS=192.168.0.30
```

然后运行下列命令:

```bash
# 设置这两个服务开机自启，且现在也启动
systemctl enable --now systemd-networkd.service
systemctl enable --now systemd-resolved.service

# 如果之前已经启动了这两个服务，则重启
systemctl restart systemd-networkd.service
systemctl restart systemd-resolved.service
```

运行完，你会发现已经能联网了，哪怕你在networkmanager把eth0断开了，电脑依然有网络，所有的流量都会通过macproxy发送给容器的macnet，再转发到局域网的路由，最后到外部。

但是我发现一个问题，macproxy作为新建的macvlan网络，它经由macnet到互联网的过程中dns有问题，只能访问解析国内的网站，steam等网站都不发解析。我在网上查到有类似问题，但信息本身就十分缺失，无法定位错误。我尝试在openwrt内部正确设置dns，但是依然无效。

可以先尝试宿主机仅利用macproxy连接访问互联网，如果同样的dns问题，我的方案是在本机也同时使用eth0连接网络，正常情况下，`ip r`可以看到两个default路由，macproxy的路由优先级高于eth0，因此会优先走容器代理，而经过容器代理无法正常访问的网站，则会通过eth0直接访问。这么做的坏处是，如果你有同样的dns问题，利用openwrt代理宿主机访问外网是不行的，我的目的只是使用雷神加速器，我的代理依然是在宿主机上运行软件代理特定的应用。


## 使用

下面说一下如何使用雷神加速器的openwrt插件：

1. 手机和电脑同处一个网络，且手机的wifi设置网关和DNS为openwrt的ip。
2. 访问openwrt的网络管理界面（直接访问openwrt的ip），打开**upnp与nat-pmp服务**。
3. 雷神加速器app->硬件加速器->安装路由器插件
4. 在openwrt的雷神加速器的插件页面，查看设备，将电脑对应的mac地址的设备，设置为windows或steamdeck
5. 手机上选择电脑游戏加速，加速想要的游戏

## 总结

本文主要利用的是macvlan使得容器在局域网内可见，随后的一系列措施都是在解决macvlan带来的问题。因此如果决定要选择这个方案，首先应该考虑双网卡，随后考虑使用veth pair连接宿主机和docker（考虑了这个方案，但没有测试），最后再尝试在宿主机建立另一个macvlan来访问这个macvlan。

梳理文章的脉络：
1. 使用docker跑了openwrt，利用macvlan网络使得局域网能够访问到容器。
2. 发现macvlan的容器唯独不可以和宿主机通信，尝试解决
3. 在宿主机建立另一个macvlan，利用这个macvlan访问容器
4. 发现利用macvlan访问容器会有莫名的dns问题
5. 在利用macvlan连接到容器的同时，也使用eth0作为备用网络。虽然未经手动设置，但它们的路由优先级应该是macvlan>eth0

所以，在第一步，可以选择使用虚拟机而不是docker，就没有这么多麻烦事。在第三步，可以尝试使用双网卡或veth pair来解决宿主机与容器的通信问题。在最后一步，需要确认macvlan的路由优先级大于eth0。 前前后后折腾了一周，我也已经使用了大半个月了，但一直没时间记录，后面有空还会改进这篇文章。

## 参考连接

1. [使用systemd开机自动打开指定网卡的混杂模式](https://wiki.archlinuxcn.org/wiki/%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE#%E6%B7%B7%E6%9D%82%E6%A8%A1%E5%BC%8F)
2. [systemd-networkd#3.3 MACVLAN Bridge](https://wiki.archlinux.org/title/Systemd-networkd)
3. [openwrt容器镜像](https://hub.docker.com/r/piaoyizy/openwrt-x86)
4. [使用macvlan访问docker的macvlan](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/)
5. [OpenWrt 雷神加速器插件管理器使用指南](https://www.miaoer.xyz/posts/blog/openwrt-leigodacc-manager)